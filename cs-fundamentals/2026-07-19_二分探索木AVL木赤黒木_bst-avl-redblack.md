# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 根 < 右」の不変条件を持つ木構造で、ソート済みデータの検索・挿入・削除を効率的に行う。
AVL木・赤黒木はBSTに自動バランス調整を加えた発展形で、最悪ケースでもO(log n)を保証する。
RDBのB-Treeインデックスやlibcのmalloc実装にも赤黒木の考え方が使われており、DB設計やメモリ管理の理解に直結する。

---

## 仕組みの要点

### 二分探索木（BST）

- ノードは `(key, left, right)` を持つ
- 検索: 根から「小さければ左、大きければ右」を繰り返すだけ
- 問題点: 昇順に挿入すると右に伸びる「偏り木」になりO(n)に劣化

```
  挿入順: 1, 2, 3, 4
  1
   \
    2
     \
      3  ← 線形リストと同じ
       \
        4
```

### AVL木（高さバランス木）

- 各ノードで「左右の高さ差 ≤ 1」を維持（バランス係数）
- 挿入・削除後に違反があれば**回転（rotation）**で修正
  - 右回転 / 左回転 / 左右回転 / 右左回転の4パターン
- 高さが常にO(log n)になるため検索も常にO(log n)
- 赤黒木より回転回数が多い → **読み取り頻度が高い用途**に向く

### 赤黒木（Red-Black Tree）

- 各ノードに色（赤/黒）を付加して以下の条件を維持:
  1. 根は黒
  2. 赤ノードの子は黒（赤が連続しない）
  3. どの根→葉パスも同じ黒ノード数（黒高さが等しい）
- AVL木より緩い制約なので回転回数が少ない → **挿入・削除頻度が高い用途**に向く
- Linux カーネルのメモリ管理、Java の `TreeMap`、C++ の `std::map` で採用

---

## 計算量

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|-------------|-------|--------|
| 検索 | O(log n)   | O(n)        | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)        | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)        | O(log n) | O(log n) |
| 回転回数（挿入時） | — | — | 最大2回 | 最大2回 |
| 回転回数（削除時） | — | — | O(log n)回 | 最大3回 |

---

## コード例（Python: BST の検索と挿入）

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None

class BST:
    def insert(self, root, key):
        if root is None:
            return Node(key)
        if key < root.key:
            root.left = self.insert(root.left, key)
        elif key > root.key:
            root.right = self.insert(root.right, key)
        return root

    def search(self, root, key):
        if root is None or root.key == key:
            return root
        if key < root.key:
            return self.search(root.left, key)
        return self.search(root.right, key)
```

---

## よくある誤解・落とし穴

- **BSTは常にO(log n)ではない** — ソート済みデータを挿入すると最悪O(n)。本番ではAVL木か赤黒木を使う
- **AVL木 vs 赤黒木の使い分けを混同する** — AVL木は厳密なバランスで検索が速いが挿入・削除の回転コストが高め。頻繁な書き込みには赤黒木
- **Pythonの `sortedcontainers.SortedList`** は内部的にBSTではなくSkip List系だが同等の計算量を持つ
- **DB の B-Tree はBSTとは別物** — B-Tree はディスクI/O最小化のためノードが複数キーを持つ（後日トピック26で詳述）

---

## 実務での使いどころ

| 場面 | 関連 |
|------|------|
| **FastAPI + Neon** のクエリ | PostgreSQLのインデックスは内部的にB-Tree（赤黒木の考え方の派生）を使用。`ORDER BY` + `LIMIT` が速いのはこの構造のため |
| **Cloud Run** のルーティング | リバースプロキシが多数のルールをBST系構造で管理し、O(log n)でマッチング |
| **Python コード** | `sortedcontainers.SortedList` で範囲検索が必要な場合（ハッシュテーブルは範囲検索不可）に使う |
| **API のレート制限** | スライディングウィンドウの実装でソート済みセット（赤黒木ベース）が有効 |
