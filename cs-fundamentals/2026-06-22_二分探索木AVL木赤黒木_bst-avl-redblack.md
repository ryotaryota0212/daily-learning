# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の性質を持つ木構造で、検索・挿入・削除をO(log n)で行える。
ただし入力順によっては木が偏り（最悪O(n)）、これを防ぐために**自己平衡木**（AVL木・赤黒木）が登場した。
PythonのsortedcontainersやPostgreSQLのB-Treeインデックスの思想的ベースになる。

---

## 仕組みの要点

### 二分探索木（BST）
- ノードは「値・左子・右子」を持つ
- 挿入: ルートから大小比較して降りていき、空きに置く
- 検索: 目標値と比較しながら左右を辿る
- 削除: 子が2つの場合は「中順後継者（右部分木の最小値）」と交換してから削除
- **問題点**: ソート済みデータを順に挿入すると線形リストになり O(n)

### AVL木
- 各ノードに「左右の高さの差（バランス因子）」を管理
- バランス因子が ±2 になったら**回転**で修正
  - 右回転・左回転・左右回転・右左回転の4種類
- 挿入・削除のたびにバランスを厳密に保つ → 検索は高速だが挿入/削除コストが高め

### 赤黒木
- ノードに「赤/黒」の色を付け、以下の制約でほぼ平衡を保つ:
  1. ルートは黒
  2. 赤ノードの子は必ず黒
  3. 任意ノードからNILまでの黒ノード数は同じ（黒高さ）
- AVL木より厳密でないが回転回数が少なく、**挿入/削除が速い**
- Pythonの `dict`（CPython 3.6+）、Java の `TreeMap`、Linux カーネルのスケジューラで使用

---

## 計算量まとめ

| 操作       | BST（平均）| BST（最悪）| AVL木     | 赤黒木    |
|----------|----------|----------|---------|---------|
| 検索       | O(log n) | O(n)     | O(log n)| O(log n)|
| 挿入       | O(log n) | O(n)     | O(log n)| O(log n)|
| 削除       | O(log n) | O(n)     | O(log n)| O(log n)|
| 空間       | O(n)     | O(n)     | O(n)    | O(n)    |

- **AVL木** は高さが厳密に ≤ 1.44 log n → 検索最速
- **赤黒木** は高さが ≤ 2 log n → 挿入/削除のバランスが良い

---

## コード例（BST の基本操作）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self): self.root = None

    def insert(self, val):
        def _ins(node, val):
            if not node: return Node(val)
            if val < node.val: node.left = _ins(node.left, val)
            elif val > node.val: node.right = _ins(node.right, val)
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

- **「BSTは常にO(log n)」は誤り** — 平衡が保たれている場合のみ。昇順挿入で最悪O(n)になる
- **AVL木の回転は複雑に見えるが本質は単純** — 親子関係のポインタ付け替えだけ
- **赤黒木の「色」は高さのプロキシ** — 色そのものに意味はなく、黒高さの均一性が目的
- **Python に BST 標準実装はない** — `sortedcontainers.SortedList` や `heapq` で代替する場面が多い

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run スタック）

- **Neon (PostgreSQL) のインデックス**: 内部はB-Tree（BSTの多分木拡張）。`EXPLAIN ANALYZE` で Index Scan が出れば木の二分探索が動いている
- **範囲検索が速い理由**: ハッシュインデックスと違いBSTは順序を保持するため `WHERE created_at BETWEEN ...` に有効
- **FastAPI のルーティング**: フレームワーク内部のルート探索にトライ木（BSTの派生）を使う実装がある
- **個人開発での直接利用場面**: リーダーボードや時系列集計など「順序付き集合の範囲取得」が必要な場合は SortedList が BST の恩恵をそのまま受ける
