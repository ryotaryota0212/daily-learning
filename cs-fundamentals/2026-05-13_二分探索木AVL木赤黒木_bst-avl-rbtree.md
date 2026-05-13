# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 根 < 右」の順序を保つ木構造で、検索・挿入・削除をO(log n)で行える。
しかし偏りが生じるとO(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用では使われる。
PythonのSortedContainers、JavaのTreeMap、PostgreSQLのB-Treeインデックスなどに応用されている。

---

## 仕組みの要点

### 二分探索木（BST）

- ノードは「左部分木 ≤ 自分 < 右部分木」を満たす
- 検索：根から大小比較しながら下に辿る
- 挿入：検索と同じ経路で空いた葉に追加
- **問題点**：昇順データを挿入すると右に一直線になり高さO(n)になる

```
挿入順: 1,2,3,4,5 の場合（最悪ケース）
1
 \
  2
   \
    3  ← リストと同等の性能に劣化
```

### AVL木（高さバランス木）

- 各ノードで「左右の高さの差 ≤ 1」を維持
- 違反時に**回転**（左回転・右回転・二重回転）で修正
- 高さが常にO(log n)に保たれる
- 挿入・削除のたびに回転コストがかかるため、**読み取り多め**の用途に向く

### 赤黒木

- 各ノードを赤か黒に色付けし、以下の制約で緩やかにバランスを保つ
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数が等しい（黒高さ一定）
- AVL木より回転回数が少ない → **挿入・削除が多い**用途に向く
- C++ `std::map`、Java `TreeMap`、Linux カーネルのスケジューラに使用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|-------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |

- AVL木の高さ ≤ 1.44 log₂(n+2)
- 赤黒木の高さ ≤ 2 log₂(n+1)（AVLより若干高いが回転が少ない）

---

## コード例（Python：シンプルなBST検索）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

def insert(root, val):
    if not root:
        return Node(val)
    if val < root.val:
        root.left = insert(root.left, val)
    else:
        root.right = insert(root.right, val)
    return root

def search(root, val):
    if not root or root.val == val:
        return root
    if val < root.val:
        return search(root.left, val)
    return search(root.right, val)
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ランダム挿入なら平均O(log n)だが、ソート済みデータでO(n)に劣化する
- **AVL木が常に赤黒木より速いわけではない** — 読み取り速度はAVLが有利だが、書き込みが多い場合は赤黒木の回転削減が効く
- **Pythonの`dict`はBSTではない** — ハッシュテーブル。順序付き辞書が必要なら`sortedcontainers.SortedDict`（内部はB-Tree的な実装）
- **削除は挿入より複雑** — 子が2つのノード削除は「中順後継者（右部分木の最小値）」で置き換える処理が必要

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

| 場面 | 関連 |
|------|------|
| **PostgreSQL（Neon）のインデックス** | B-Treeインデックスは赤黒木の考え方を多分木に拡張したもの。`CREATE INDEX`の仕組みの理解に直結 |
| **範囲クエリの最適化** | `WHERE id BETWEEN 100 AND 200` はBSTの中順探索と同じ原理。インデックスが効く理由を説明できる |
| **APIのレスポンスソート** | データ取得後のメモリ上ソートより、DBのインデックスを活用した方が高速な理由の根拠になる |
| **キャッシュのTTL管理** | 期限切れキャッシュを効率的に探すには順序付きデータ構造が有効 |
