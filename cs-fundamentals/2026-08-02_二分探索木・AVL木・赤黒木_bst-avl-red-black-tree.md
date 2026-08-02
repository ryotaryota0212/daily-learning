# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の性質を持つ木構造で、探索・挿入・削除をO(log n)で行える。
しかし偏ったデータを挿入するとO(n)に劣化する。この問題を解決するのが**自己平衡木**（AVL木・赤黒木）。
実務ではRDBのインデックス（B-Tree）、Java の `TreeMap`、C++ の `std::map` などに応用される。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `left < self < right` を満たす
- 探索: 根から値を比較し左右に降りる
- 挿入: 探索と同じ経路で末尾に追加
- 削除: 対象ノードの後継者（右部分木の最小値）で置き換える
- **問題点**: `[1,2,3,4,5]` を順挿入すると右に偏ったリスト状になりO(n)

```
挿入: 1, 2, 3, 4 の場合
1
 \
  2
   \
    3
     \
      4  ← 線形リストと同じ
```

### AVL木

- 各ノードで `|左高さ - 右高さ| ≤ 1` を常に保つ（高さバランス条件）
- 挿入・削除後に高さを再計算し、違反箇所で**回転（Rotation）**を実行
- 回転の種類: LL回転、RR回転、LR回転、RL回転
- 高さが常に `O(log n)` に保たれるため探索が保証される
- **欠点**: 回転コストが高く、書き込み多用途では赤黒木より遅い

### 赤黒木

- 各ノードに「赤」か「黒」の色を付ける
- **5つの性質**:
  1. 各ノードは赤か黒
  2. 根は黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 根から全葉までの黒ノード数が等しい（黒高さ一定）
- 上記性質により最長経路 ≤ 最短経路 × 2 → O(log n) 保証
- AVL木より緩いバランスだが**回転回数の上限が定数**（挿入: 最大2回、削除: 最大3回）
- Linux カーネルのプロセス管理、Java `TreeMap`、C++ `std::map` に採用

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木   | 赤黒木  |
|----------|------------|------------|---------|---------|
| 探索     | O(log n)   | O(n)       | O(log n)| O(log n)|
| 挿入     | O(log n)   | O(n)       | O(log n)| O(log n)|
| 削除     | O(log n)   | O(n)       | O(log n)| O(log n)|
| 回転回数 | -          | -          | O(log n)| O(1)    |

- AVL木は**探索重視**（より厳密なバランス = 木が低い）
- 赤黒木は**挿入・削除重視**（回転コスト低い）

---

## コード例（Python: BSTの基本）

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
            else:
                node.right = _insert(node.right, val)
            return node
        self.root = _insert(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val:
                return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ランダムデータなら平均O(log n)だが、ソート済み入力でO(n)に劣化する
- **AVL木 ≠ 最速** — 探索は高速だが、書き込みが多いワークロードでは赤黒木が勝ることが多い
- **赤黒木の実装は複雑** — 実務では自分で実装せず言語標準ライブラリを使う（`sortedcontainers`、`TreeMap`など）
- **RDBのインデックスはB-Tree（B+Tree）** — 二分木ではなく多分木。ページ（ディスクブロック）単位でデータを保持しI/Oを最小化する設計

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX` で作られるB+TreeはBSTの拡張版。EXPLAINで`Index Scan`が出れば木を降りている
- **範囲クエリ**: `WHERE created_at BETWEEN ...` が速いのはB+Treeが順序を保持するから（ハッシュインデックスでは不可）
- **FastAPIのルーティング**: パスパラメータの探索に木構造が内部利用されているケースがある
- **ソート済みデータのAPI**: フロントに返すデータを都度ソートするよりBSTベースの構造を維持する方が効率的（挿入の多いランキング機能など）
- **学習ポイント**: `EXPLAIN ANALYZE` を読む際、木の深さとコストの関係を理解していると実行計画が直感的に把握できる
