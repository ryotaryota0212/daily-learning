# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という順序付きツリー構造で、探索・挿入・削除をO(log n)で行える。しかし偏り（退化）があるとO(n)まで劣化する。AVL木と赤黒木はこの問題を「自己平衡」によって解決する。インデックス実装・優先度付きマップ・DBのB-Tree理解の基礎になる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左サブツリー < 自身 < 右サブツリー」を保証
- 探索：値を比較しながら左右を辿るだけ
- **問題：** ソート済みデータを順に挿入すると直線状（O(n) list）になる

```
  [1]
    \
    [2]
      \
      [3]  ← 退化した BST（これはただのリスト）
```

### AVL木

- 各ノードで「左右の高さの差（バランス因子）が ±1 以内」を常に維持
- 挿入・削除後に**回転（rotation）**で再平衡化
  - 右回転、左回転、左右回転、右左回転の4パターン
- 利点：常に高さが O(log n) で探索が速い
- 欠点：回転頻度が高くなり挿入・削除コストがやや高い

### 赤黒木

- 各ノードを「赤」か「黒」に色付け、以下の4条件を維持:
  1. 根はブラック
  2. 赤ノードの子は必ずブラック（赤が連続しない）
  3. どの葉から根への黒ノード数は同じ（黒高さ一定）
  4. 葉（NILノード）はブラック
- AVL木より制約が緩い → 回転回数が少ない → 挿入・削除が速い
- 探索はAVL木より若干遅い（高さの上限が2倍程度）

---

## 計算量・パフォーマンス特性

| 操作 | BST（最良） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n) | O(n) |

- AVL木：探索が多い場合に有利（高さが最小）
- 赤黒木：挿入・削除が多い場合に有利（回転が少ない）
- **PythonのsortedcontainersやJavaのTreeMap・TreeSetは赤黒木ベース**

---

## コード例（Python：BST の探索・挿入）

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

- **「BSTは常にO(log n)」は誤り** — 平衡が崩れるとO(n)に退化する
- **AVL木は万能ではない** — 書き込みが多いワークロードでは赤黒木の方が速い
- **赤黒木の「黒高さ一定」が鍵** — これが最悪高さ 2log(n) を保証する
- Python標準ライブラリにはBSTの実装がない — `sortedcontainers.SortedList` を使う

---

## 実務での使いどころ

| 場面 | 関連 |
|------|------|
| FastAPI レスポンスのソート済みデータ管理 | `SortedList` で範囲クエリをO(log n)に |
| Neon(PostgreSQL) のインデックス | B-Tree（BST を一般化したもの）が内部実装 |
| Cloud Run のレートリミット | 順序付きセットで「直近N秒のリクエスト数」を管理 |
| DB の `ORDER BY` + `RANGE` クエリ高速化 | B-Tree インデックスの理解につながる |

BST/AVL/赤黒木の原理を理解することで、PostgreSQLのEXPLAIN出力の「Index Scan」がなぜ速いかが直感的にわかるようになる。
