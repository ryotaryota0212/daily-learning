# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

**二分探索木（BST）** は「左 < 親 < 右」という順序制約を持つ木構造で、探索・挿入・削除をO(log n)で実現する。しかし普通のBSTは偏りが生じるとO(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用に使われる。PythonのSortedContainers、JavaのTreeMap、PostgreSQLのB-Treeインデックスはいずれもこの系譜。順序付きデータの高速操作が必要なときに選ぶ構造。

---

## 仕組みの要点

### 二分探索木（BST）の基本

```
     8
    / \
   3   10
  / \    \
 1   6    14
    / \
   4   7
```

- **探索**: ルートから「小さければ左、大きければ右」を繰り返す
- **挿入**: 探索で落ちた位置にノードを追加
- **削除**: 3パターン（葉、子1つ、子2つ）。子2つの場合は「中順後継者」で置換
- **問題**: 昇順にinsertすると線形リストになりO(n)に劣化

### AVL木（高さ平衡木）

- **平衡条件**: 全ノードで `|左の高さ - 右の高さ| ≤ 1`
- **回転で修復**: 挿入・削除後に違反が起きたら「右回転・左回転・左右回転・右左回転」で修正
- **高さ保証**: n個のノードで高さ ≤ 1.44 × log₂n
- **特徴**: 非常に厳格な平衡 → 探索が速い。挿入・削除は回転が多め

### 赤黒木（Red-Black Tree）

5つの規則で平衡を保つ：
1. 各ノードは赤か黒
2. ルートは黒
3. 赤ノードの子は黒（赤が連続しない）
4. 任意のノードから葉までの「黒ノード数」は同じ（黒高さ一定）
5. 全ての葉（NIL）は黒

- **高さ保証**: n個のノードで高さ ≤ 2 × log₂(n+1)
- **AVL木との比較**: 平衡がやや緩い → 挿入・削除の回転が少ない
- **実用例**: Linux Kernel、Java TreeMap、C++ std::map、多くのDBのインデックス

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

**AVL vs 赤黒木の選択**:
- 探索が多い → AVL木（高さが低く探索が速い）
- 挿入・削除が多い → 赤黒木（再バランスのコストが低い）
- 汎用途 → 赤黒木（ほとんどの標準ライブラリの選択）

---

## コード例（Python：BST探索の基本）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, val):
            if not node:
                return Node(val)
            if val < node.val:
                node.left = _ins(node.left, val)
            elif val > node.val:
                node.right = _ins(node.right, val)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

実用ではPythonの `sortedcontainers.SortedList`（内部はB-Tree系）を使うのが現実的。

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ソート済みデータを順番にinsertすると最悪O(n)。実装には必ず自己平衡木を使う
- **「赤黒木はAVL木より優れている」も誤り**: 探索コストはAVL木の方が低い。用途次第
- **削除が難しい**: 子が2つあるノードの削除は中順後継者（右部分木の最小値）を使う。面接でよく問われる
- **回転の方向混乱**: 「右回転」は親を右に倒す操作で、左の子が新しい親になる。図を書いて確認すること

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run スタック）

- **Neon（PostgreSQL）**: `CREATE INDEX` で作られるB-Treeは赤黒木の発展形。`EXPLAIN ANALYZE` でインデックスが使われているか確認するとき、「どう探索しているか」の理解に直結する
- **範囲クエリ**: `WHERE price BETWEEN 100 AND 500` がインデックスで効率的なのはBSTの性質（順序保持）のおかげ。ハッシュインデックスは範囲クエリに使えない
- **FastAPI のルーティング**: 内部的にはトライ木やハッシュマップだが、順序付きルートが必要な場面ではBST系が使われる
- **アルゴリズム問題**: LeetCodeのTree系問題（98: Validate BST、230: Kth Smallest等）は面接頻出。BST性質を使ったin-order traversalのパターンを押さえる
