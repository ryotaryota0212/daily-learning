# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序不変条件を持つ木構造。検索・挿入・削除がO(log n)で動く**はず**だが、挿入順によっては縦長になりO(n)に退化する。この退化を防ぐのが自己平衡木（AVL木・赤黒木）。実務では直接使う機会は少ないが、RDBMSのB-Treeインデックスや標準ライブラリのSortedSetはこれらの派生。「なぜインデックスが速いか」を理解する基礎になる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノード: `value`, `left`, `right`
- 不変条件: `left.value < node.value < right.value`
- 検索: 根から比較して左右に降りる → O(h)、hは木の高さ
- 退化例: [1, 2, 3, 4, 5] を順に挿入 → 片側に伸びる線形リスト → O(n)

### AVL木（高さ平衡木）

- 各ノードの左右部分木の高さ差（バランス因子）を **-1, 0, +1** に維持
- 挿入・削除後にバランス因子が崩れたら**回転**で修正
  - 単回転（右回転 / 左回転）
  - 二重回転（左右回転 / 右左回転）
- 高さは常に O(log n) → 検索保証あり
- 欠点: 挿入・削除ごとに再バランスが頻繁 → 書き込み多い場合はオーバーヘッド

### 赤黒木

- 各ノードに**赤 or 黒**の色属性を付加
- 4つの不変条件:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数は等しい（黒高さ一定）
  4. 葉（NIL）は黒
- これにより高さは `2 * log(n+1)` 以下に抑えられる
- AVL木より回転回数が少なく、**挿入・削除が高速**
- Linux カーネルのスケジューラ（CFS）、Java の `TreeMap`、C++ の `std::map` で採用

---

## 計算量比較

| 操作 | BST(平均) | BST(最悪) | AVL木 | 赤黒木 |
|------|-----------|-----------|-------|--------|
| 検索 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 挿入 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 削除 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 回転回数(挿入) | - | - | ≤2 | ≤2 |
| 回転回数(削除) | - | - | O(log n) | ≤3 |

- **検索が多い**: AVL木（高さが厳密に低い）
- **挿入・削除が多い**: 赤黒木（再バランスコスト小）

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

- **「BSTは常にO(log n)」は誤り**: ソート済みデータを順挿入すると O(n) に退化
- **AVL木 = 最速ではない**: 削除時の回転が多く、書き込み多用途では赤黒木が有利
- **Pythonの `sortedcontainers.SortedList`**: 実装はB-Treeに近い構造。純粋なBSTではない
- **高さとノード数の混同**: n個のノードを持つ完全二分木の高さは `⌊log₂n⌋`

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **NeonのB-Treeインデックス**: PostgreSQLのインデックスはB-Tree（BSTの多分木拡張）。`CREATE INDEX`の仕組みを理解するとクエリチューニングに直結
- **ソート済み範囲検索**: `BETWEEN`や`ORDER BY`が速いのはB-Treeが順序を保持するため
- **FastAPIのルーティング**: Radix tree（Trie木）を使ったルート解決。木構造の応用
- **Cloud RunのオートスケールキューにPriorityQueue**: ヒープ（木構造）でO(log n)スケジューリング

### 実装より「なぜ速いか」の理解が重要

- インデックスを貼るべき列の判断 → B-Treeの特性理解から
- `EXPLAIN ANALYZE`でIndex Scanが使われる条件の把握
- ORMが生成するN+1クエリの問題と、適切なJOINへの置き換え判断
