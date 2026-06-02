# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「検索・挿入・削除をO(log n)で行える」データ構造の基本形。  
ただし素朴なBSTは偏りが生じるとO(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用で使われる。  
PythonのsortedcontainersやDBのB-Treeインデックスの根本原理でもあり、「なぜインデックスが速いか」を理解する出発点になる。

---

## 仕組みの要点

### 二分探索木（BST）の基本ルール

```
       8
      / \
     3   10
    / \    \
   1   6    14
      / \
     4   7
```

- 各ノードの左部分木は「すべて小さい値」、右部分木は「すべて大きい値」
- 中順走査（左→根→右）をすると昇順ソート済みの列が得られる
- 検索・挿入・削除はすべて根から辿るだけ → 高さ h の木なら O(h)

### 偏りの問題

- 昇順データを順に挿入すると右にだけ伸びる連結リスト化 → O(n)
- これを防ぐのが**自己平衡木**

### AVL木

- 各ノードで「左右部分木の高さの差（バランス因子）≤ 1」を常に維持
- 挿入・削除後に回転操作（単回転・二重回転）でバランスを回復
- **左回転・右回転のイメージ:**

```
    y              x
   / \    →      / \
  x   C         A   y
 / \               / \
A   B             B   C
```

- 検索が多い用途では赤黒木より優秀（より厳密にバランス）

### 赤黒木

- 各ノードに「赤/黒」の色を付け、5つの性質を保つ
  1. 全ノードは赤か黒
  2. 根は黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの「黒ノード数」は一定（黒高さ）
- 高さは最大 2log(n+1) → AVL木より少し緩いが挿入・削除の回転回数が少ない
- **Pythonの`sortedcontainers`、Java/C++の`TreeMap`/`std::map`が採用**

---

## 計算量・パフォーマンス特性

| 操作 | 素のBST（平均） | 素のBST（最悪） | AVL木 | 赤黒木 |
|------|---------------|----------------|-------|--------|
| 検索 | O(log n)      | O(n)           | O(log n) | O(log n) |
| 挿入 | O(log n)      | O(n)           | O(log n) | O(log n) |
| 削除 | O(log n)      | O(n)           | O(log n) | O(log n) |
| 空間 | O(n)          | O(n)           | O(n)  | O(n) |

- 赤黒木は挿入・削除時の回転数が**最大2〜3回**（AVL木は O(log n) 回）
- 読み取り重視 → AVL木、書き込み重視 → 赤黒木が有利

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
        def _insert(node, val):
            if not node:
                return Node(val)
            if val < node.val:
                node.left = _insert(node.left, val)
            elif val > node.val:
                node.right = _insert(node.right, val)
            return node
        self.root = _insert(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ランダムデータなら平均O(log n)だが、ソート済みデータでO(n)になる
- **AVL木・赤黒木は「完全平衡」ではない** — 高さが最小になるわけではなく、上界を保証するだけ
- **削除が一番複雑** — 削除するノードに子が2つある場合、中順後継ノードで置き換える必要がある
- **Pythonの`dict`や`set`はBSTでなくハッシュテーブル** — ソート順の走査が必要なら`SortedList`（赤黒木ベース）を使う

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **NeonのB-Treeインデックス**: PostgreSQLのインデックスはB-Tree（BSTの多分木拡張）。`WHERE id = ?` や `ORDER BY created_at` が速い理由がここにある。`EXPLAIN ANALYZE` で "Index Scan" が出たらB-Treeが使われている証拠
- **範囲クエリに強い**: ハッシュインデックスと違い `BETWEEN`・`>` `<` もO(log n)で処理できる。Neonでタイムスタンプ範囲検索するなら`BTREE`インデックスが最適
- **ソート済み走査**: 中順走査 = ソート済み出力という性質が、DBの`ORDER BY`コスト削減に直結
- **Cloud Runのメモリ上キャッシュ**: インメモリでソート済みコレクションが必要なときは`sortedcontainers.SortedList`（赤黒木ベース）が有効
