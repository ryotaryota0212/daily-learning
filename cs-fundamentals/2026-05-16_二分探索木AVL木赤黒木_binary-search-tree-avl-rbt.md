# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

順序付きデータの検索・挿入・削除を効率化するデータ構造。ハッシュテーブルが「完全一致」に強いのに対し、**BST系は範囲検索・順序走査**に強い。PostgreSQLのB-Treeインデックスや、言語標準ライブラリ（Python `sortedcontainers`、Java `TreeMap`）の内部で使われており、データベース設計やAPIのソート・フィルタ処理の計算量を理解するために不可欠。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは「左 < 自分 < 右」の不変条件を持つ
- 検索・挿入・削除はすべて根から辿るだけ
- **問題**: 挿入順が偏ると木が線形に退化（最悪 O(n)）

```
挿入順: 5, 3, 7, 1, 4
    5
   / \
  3   7
 / \
1   4   ← バランスが取れている

挿入順: 1, 2, 3, 4, 5
1
 \
  2
   \
    3  ← 連結リストと同じ。O(n)に退化
```

### AVL木
- 各ノードで `|左の高さ - 右の高さ| ≤ 1` を強制
- 挿入・削除後に**回転操作**（左回転・右回転・二重回転）で再バランス
- BSTより厳密にバランスを保つ → 検索は高速、挿入・削除は回転コストあり

### 赤黒木（Red-Black Tree）
- ノードを赤/黒に色付けし、以下の4条件で緩やかなバランスを保つ
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 根からどの葉までも黒ノード数が同じ
  4. 葉（NILノード）は黒
- AVL木より回転が少ない → **挿入・削除が多いユースケースで有利**
- Linux カーネルのスケジューラ、C++ `std::map` の内部実装で採用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 範囲検索 | O(k + log n) | O(n) | O(k + log n) | O(k + log n) |

- AVL木: 高さ ≤ 1.44 log₂(n+2) ← 最も厳密
- 赤黒木: 高さ ≤ 2 log₂(n+1) ← 少し緩いが回転操作が少ない

---

## コード例（Python: BST の基本操作）

```python
class Node:
    def __init__(self, val):
        self.val, self.left, self.right = val, None, None

class BST:
    def __init__(self): self.root = None

    def insert(self, val, node=None):
        if node is None:
            if self.root is None: self.root = Node(val); return
            node = self.root
        if val < node.val:
            if node.left is None: node.left = Node(val)
            else: self.insert(val, node.left)
        else:
            if node.right is None: node.right = Node(val)
            else: self.insert(val, node.right)

    def inorder(self, node=None):  # ソート済み出力: O(n)
        if node is None: node = self.root
        if node.left: yield from self.inorder(node.left)
        yield node.val
        if node.right: yield from self.inorder(node.right)
```

---

## よくある誤解・落とし穴

- **「BST = 常に O(log n)」は誤り**: ランダム挿入なら平均 O(log n) だが、ソート済みデータを挿入すると O(n) に退化する
- **AVL木が常に最速とは限らない**: 挿入・削除が頻繁な場合、回転コストで赤黒木に劣ることがある
- **Python に組み込みの BST はない**: `sortedcontainers.SortedList` や `bisect` モジュールで代替（内部は配列ベース）
- **インオーダー走査でソート済み出力**: これは BSTの本質的な性質で、範囲クエリに応用できる

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **Neon（PostgreSQL）のインデックス**: B-Tree は赤黒木の考え方をディスク向けに拡張した構造。`WHERE created_at BETWEEN ...` のような範囲検索が速い理由がここにある
- **APIのソート・フィルタ**: DB に任せる前に、インメモリで小規模なソート済み集合を管理したい場面で `SortedList` が使える
- **ジョブキュー / スケジューラ**: 優先度付きキュー（次章: ヒープ）の代替として、同一優先度内で順序を保ちたい場合に BST が有効
- **計算量の見積もり**: Neon の `EXPLAIN ANALYZE` で Index Scan が出たとき、その O(log n) コストを正確に理解できる
