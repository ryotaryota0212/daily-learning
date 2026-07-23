# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という不変条件によって、挿入・検索・削除をすべて O(log n) で行える木構造。ただし偏りが生じると O(n) に劣化するため、実務ではAVL木や赤黒木などの「自己平衡木」が使われる。データベースのインデックス（B-Tree）や言語標準ライブラリの `map` / `set` はこれらの変形実装。「なぜ O(log n) を保証できるか」を理解することで、パフォーマンス問題の根本原因を掴める。

---

## 仕組みの要点

### 二分探索木（BST）

- **不変条件**: 各ノードについて `左部分木のすべて < ノード < 右部分木のすべて`
- **検索**: ルートから「小さければ左・大きければ右」を繰り返す
- **挿入**: 検索と同じ経路で末端に追加
- **削除**: 子が2つある場合は「中順後継ノード」(右部分木の最小値) で置換
- **問題点**: ソート済みデータを挿入すると一直線になり O(n) に劣化

```
通常のBST（バランス良好）     偏ったBST（ソート順で挿入）
        5                      1
       / \                      \
      3   7                      2
     / \   \                      \
    2   4   8                      3  ← 検索O(n)
```

### AVL木

- BSTに「**各ノードの左右の高さの差（バランス因子）が -1, 0, 1 であること**」を付加
- 挿入・削除後に違反したら**回転（rotation）**で修正
  - **右回転**: 左が重いとき（左の左ケース）
  - **左回転**: 右が重いとき（右の右ケース）
  - **左右回転・右左回転**: 折れ曲がりケース
- 高さは常に O(log n) に保たれる

```
右回転の例（左の左ケース）:
    z               y
   /              /   \
  y      →       x     z
 /
x
```

### 赤黒木

- 各ノードに「赤」か「黒」の色を持たせ、5つの性質で**高さを 2 log n 以内**に保つ
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意ノードからすべての葉への黒ノード数は等しい（黒高さ一定）
- AVL木より**回転回数が少ない**ため挿入・削除が速い
- 実装例: C++ `std::map`, Java `TreeMap`, Linux カーネルのスケジューラ

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- **AVL木**: 高さが厳密に floor(1.44 log n) 以下 → 検索が多いなら有利
- **赤黒木**: 挿入・削除の回転が最大3回 → 書き込みが多いなら有利

---

## コード例（Python: BSTの基本操作）

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
            elif v > node.val:
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

- **「BSTは常に O(log n)」は誤り**: ランダムデータなら平均 O(log n) だが、ソート済みや逆順では O(n)。本番データがランダムかどうかは保証できない
- **AVL木と赤黒木の使い分けを誤る**: 検索頻度が高い → AVL木（より厳密にバランス）、挿入・削除頻度が高い → 赤黒木（回転コストが低い）
- **Pythonの `dict` はBSTではない**: Pythonの `dict` はハッシュテーブル。順序を保つ `SortedList` や `sortedcontainers` ライブラリが平衡木に相当
- **削除の「中順後継ノード」を忘れる**: 子が2つある削除ケースが最も実装しやすいミスポイント

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタック との関連**

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX` はB-Tree（赤黒木の派生）で実装。`WHERE id = ?` や `ORDER BY` が速いのはこれのおかげ。`EXPLAIN ANALYZE` で `Index Scan` が出ればB-Treeを使っている
- **範囲クエリ**: `BETWEEN`, `>=`, `<=` はハッシュインデックスでは使えないが、B-Treeインデックスなら O(log n + k)（k=結果件数）で効く
- **API のソート済みレスポンス**: インメモリで少量データをソートするなら Python の `heapq`（ヒープ）や `sorted()`（Timsort）で十分。大量データはDBの `ORDER BY` にインデックスを効かせる
- **Cloud Run のスケールアウト**: リクエストのルーティングや優先度管理に内部でヒープや平衡木を使うケースがあるが、アプリ層では意識不要。ただし「なぜ O(log n) か」を知っているとパフォーマンス見積もりの精度が上がる
