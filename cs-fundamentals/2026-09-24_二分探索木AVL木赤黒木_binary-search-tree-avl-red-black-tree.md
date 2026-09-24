# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の不変条件を持つ木構造で、検索・挿入・削除を平均 O(log n) で行える。しかし単純な BST はデータの挿入順によって最悪 O(n) まで劣化する。AVL 木・赤黒木はこの劣化を防ぐ**自己平衡 BST** で、データベースインデックスや言語ランタイムの内部実装に広く使われる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノード: `value`, `left`, `right` を持つ
- 探索: ルートから「値が小さければ左、大きければ右」と降りる
- 挿入: 探索と同様に空きリーフに追加
- **問題**: ソート済みデータを挿入すると右（または左）だけに伸びた「棒」になる → 高さ O(n)

```
挿入: 1, 2, 3, 4, 5
  1
   \
    2
     \
      3  ← 線形リスト化
```

### AVL 木

- **平衡条件**: 全ノードで `|左高さ - 右高さ| ≤ 1`
- 挿入・削除後に条件違反を検出したら**回転**で修正
  - 右回転 / 左回転（1回転）
  - 左右回転 / 右左回転（2回転）
- 高さが常に O(log n) に保たれる
- 検索が多いユースケースに最適（厳密な平衡 → 探索高さが最小）

### 赤黒木

- 各ノードに **赤 or 黒** の色を付加
- 不変条件:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意ノードから葉までの黒ノード数が同じ（黒高さが一定）
- AVL より平衡条件が緩い → **挿入・削除が速い**（回転回数が少ない）
- Java の `TreeMap`、Linux カーネルのスケジューラ、nginx の内部タイマー等で採用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL 木 | 赤黒木 |
|------|------------|------------|--------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転数（挿入） | - | - | 最大 2 | 最大 2 |
| 回転数（削除） | - | - | O(log n) | 最大 3 |

- **検索優先** → AVL（より厳密に平衡）
- **挿入・削除優先** → 赤黒木（回転コストが低い）

---

## コード例（Python - 基本的なBST）

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

- **「BSTは常にO(log n)」は誤り** — ソート済みや逆順データでO(n)に劣化。本番では必ず自己平衡木を使う
- **AVLは「完全平衡」ではない** — 高さの差が1まで許容される。完全平衡はコストが高すぎる
- **赤黒木の実装は複雑** — 色の引き継ぎや叔父ノードの色によって4パターン以上の場合分けが必要。実装バグを避けるため既存ライブラリを使う
- **メモリオーバーヘッド** — 各ノードにポインタ2本（左右）＋値を持つため、小さな値でも1ノードあたり数十バイト消費する。要素数が少ない場合はソート済み配列 + 二分探索の方が速い

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のB-Tree インデックス** は赤黒木と同族の自己平衡木構造を採用。`WHERE id = X` や `ORDER BY` が O(log n) で動く理由がここにある
- **Python の `sortedcontainers.SortedList`** — 内部で配列ベースのBST的構造を使用。FastAPI のリクエストハンドラ内でリアルタイムランキングや範囲検索が必要な場合に有効
- **Cloud Run のコールドスタート最適化** — 起動時に大量データをソートして保持する場合、BST より `heapq`（ヒープ）の方がシンプルで高速なケースが多い（次回トピック）
- **キャッシュの有効期限管理** — 赤黒木は「最小値の取得」が O(log n) で可能なため、TTL付きキャッシュの期限切れエントリ探索に使われる（例: Redis の Sorted Set 内部実装）
