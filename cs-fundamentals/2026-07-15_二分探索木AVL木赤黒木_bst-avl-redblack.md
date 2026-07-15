# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の不変条件を持つ木構造で、検索・挿入・削除を効率的に行える。
しかし単純なBSTは偏ったデータで最悪O(n)に劣化するため、AVL木・赤黒木などの**自己平衡木**が実用されている。
PythonのSortedContainersや各種データベースのインデックス内部で使われており、「順序を保ちながら高速に探索したい」場面で本質的な役割を担う。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードに「左サブツリーの全ノード < ノード < 右サブツリーの全ノード」を保証
- 検索：ルートから比較して左右へ降りる
- 挿入：検索して末端に追加
- **問題点**：昇順データを挿入すると木が右に伸びてリストと同じになる（高さ = n）

```
挿入順: 1, 2, 3, 4, 5
    1
     \
      2
       \
        3  ← 最悪ケースの偏り（高さ = n）
```

### AVL木

- 各ノードで「左右のサブツリーの高さ差 ≦ 1」を維持（**平衡係数**）
- 挿入・削除後に**回転操作**（左回転・右回転・二重回転）で再平衡化
- 高さが常に O(log n) に保たれる
- 回転が頻繁なためキャッシュには若干不利。読み取り多用途向き

```
不平衡検出 → 回転でリバランス
    3             2
   /    →        / \
  2             1   3
 /
1
（左回転で解消）
```

### 赤黒木

- 各ノードを「赤」or「黒」に色付け。以下の5条件を保持：
  1. 全ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意ノードから葉までの黒ノード数は等しい（黒高さ）
- AVL木より条件が緩い分、回転頻度が少なく**書き込み性能が高い**
- Linux kernelのスケジューラ（CFS）、Java TreeMap、C++ std::mapで採用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 回転 | なし       | なし       | 多い   | 少ない  |

- AVL木：高さ ≤ 1.44 log₂(n+2) を保証（より厳密に低い）
- 赤黒木：高さ ≤ 2 log₂(n+1) を保証（緩いが回転が少ない）

---

## コード例（Python: 単純BST）

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

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**。偏りが生じると最悪O(n)。ランダムデータ前提は危険
- **AVL木のほうが常に速い**わけではない。書き込みが多い場合は赤黒木のほうが有利
- **回転はノードの値を変えない**。ポインタの付け替えだけ。値の移動はない
- Pythonの`sortedcontainers.SortedList`は実装上スキップリストに近い。BSTではない

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **Neon（PostgreSQL）のB-Tree インデックス**：B-Tree は多分木だが、BSTの平衡木の考え方を拡張したもの。`CREATE INDEX`の裏側を理解するベースになる
- **範囲クエリの効率**：`WHERE created_at BETWEEN ...`はB-Treeインデックスを活用。ハッシュインデックスでは範囲検索不可。BST的な順序保持の価値がここにある
- **インメモリソート済みコレクション**：FastAPIのキャッシュや重複排除ロジックで`SortedList`（内部は平衡木系）を使うと検索O(log n)を維持できる
- **本番での重要度**：直接BSTを実装することは稀だが、DB実行計画（`EXPLAIN ANALYZE`）を読む際に「なぜインデックスが効く/効かないか」を理解するために必須の知識
