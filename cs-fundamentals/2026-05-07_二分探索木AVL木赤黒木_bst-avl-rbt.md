# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、探索・挿入・削除をO(log n)で行える。
ただし素朴なBSTは入力順次第で偏り、最悪O(n)に劣化する。
AVL木・赤黒木はそれを防ぐ**自己平衡木**で、データベースのインデックスや言語標準ライブラリの内部実装に広く使われる。

---

## 仕組みの要点

### 二分探索木（BST）基本
- 各ノードに「左サブツリー < ノード値 ≤ 右サブツリー」を保証
- 中順走査（in-order traversal）でソート済みの列が得られる
- **問題点**: ソート済みデータを挿入すると連結リストと同じ形（高さ n）になる

```
挿入順: 1, 2, 3, 4, 5 → 右に伸びる棒状の木
         1
          \
           2
            \
             3  (高さ = n, 探索 O(n))
```

### AVL木
- 各ノードで「左右の部分木の高さ差 ≤ 1」を常に維持
- 挿入・削除のたびに**回転**（左回転・右回転・LR回転・RL回転）で再平衡
- 高さが常に O(log n) に保たれる
- **特徴**: 赤黒木よりも厳格にバランスするので探索が速い。挿入・削除は回転が多め

### 赤黒木（Red-Black Tree）
- 各ノードを赤か黒に色付け、以下の制約で高さを O(log n) に保つ
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数が等しい（黒高さ一定）
- AVL木より緩い平衡 → **挿入・削除の回転回数が少ない**
- C++ `std::map`、Javaの `TreeMap`、Linuxカーネルのスケジューラで使用

---

## 計算量・パフォーマンス特性

| 操作 | 素朴BST（平均） | 素朴BST（最悪） | AVL木 | 赤黒木 |
|------|:---------:|:---------:|:-----:|:------:|
| 探索 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 挿入 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 削除 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 回転（挿入時）| — | — | 最大2回 | 最大2回 |
| 回転（削除時）| — | — | O(log n)回 | 最大3回 |

- **探索重視** → AVL木（より厳密なバランス）
- **挿入・削除重視** → 赤黒木（回転コストが小さい）

---

## コード例（Python: 素朴BSTの探索・挿入）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, v):
            if not node:
                return Node(v)
            if v < node.val:
                node.left = _ins(node.left, v)
            else:
                node.right = _ins(node.right, v)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — 自己平衡なしでは挿入順に依存する
- **AVL木の削除は重い** — 高さ差チェックが根まで伝播し、回転がO(log n)回発生しうる
- **赤黒木は「赤が半分」ではない** — 黒高さ一定が保証するのは高さ ≤ 2×黒高さ
- **Pythonの`dict`はハッシュテーブル**でBSTではない。`sortedcontainers.SortedList`等が別途必要

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタック**との関連:

- **Neon（PostgreSQL）のB-Treeインデックス**: 内部は赤黒木ではなくB-Treeだが、自己平衡の考え方は同じ。`WHERE id = ?` の高速探索はこの原理に基づく
- **範囲クエリ（BETWEEN、ORDER BY）**: BSTは中順走査で順序付きデータを得られるため、B-Treeも範囲スキャンが得意。ハッシュインデックスでは範囲クエリ不可
- **APIレスポンスのソート済みデータ**: サーバー側でSortedContainersを使うと、挿入しながらソート順を維持でき、毎回`sorted()`するよりO(log n)ずつコストを払える
- **優先度付きキュー（次回トピック）**: ヒープとBSTの使い分けを意識するとCloudRunでのジョブ管理実装が効率化される
