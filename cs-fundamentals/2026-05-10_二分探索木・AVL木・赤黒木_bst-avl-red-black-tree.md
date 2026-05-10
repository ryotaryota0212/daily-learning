# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は、検索・挿入・削除を効率よく行うための基本データ構造。  
ただし、偏ったデータを挿入すると木が深くなり O(n) に劣化する。  
AVL木・赤黒木はそれぞれ自動でバランスを保ち、最悪ケースでも O(log n) を保証する。  
DBのインデックス（B-Tree）やプログラミング言語の標準ライブラリ（`std::map`、Python の `SortedList`）の内部で使われる根幹原理。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左の子 < 自分 < 右の子」を満たす
- 中順探索（in-order traversal）でソート済みリストが得られる
- **問題**: 昇順データを挿入すると一本道になり、リスト同様の O(n) に劣化

```
挿入順: 1, 2, 3, 4, 5
      1
       \
        2
         \
          3  ← 最悪ケース（ただのリスト）
```

### AVL木

- **高さバランス条件**: 各ノードで「左右の部分木の高さ差 ≤ 1」
- 条件違反時に**回転（rotation）**で修正
  - 右回転 / 左回転 / 左右回転 / 右左回転 の4パターン
- 挿入・削除のたびに回転が発生 → 赤黒木より定数係数が大きい
- 検索が多い場合に有利（より厳密なバランス保証）

### 赤黒木

- 各ノードに「赤」か「黒」の色を付ける
- 守るべき5つの性質:
  1. 全ノードは赤か黒
  2. ルートは黒
  3. 葉（NILノード）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの黒ノード数は等しい（黒高さ）
- AVL木より緩いバランス → 回転頻度が低く、挿入・削除が速い
- Linux カーネル、Java の `TreeMap`、C++ の `std::map` で採用

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木    | 赤黒木   |
|----------|-------------|-------------|----------|----------|
| 検索     | O(log n)    | O(n)        | O(log n) | O(log n) |
| 挿入     | O(log n)    | O(n)        | O(log n) | O(log n) |
| 削除     | O(log n)    | O(n)        | O(log n) | O(log n) |
| 空間     | O(n)        | O(n)        | O(n)     | O(n)     |

- AVL木は高さが最大 1.44 log n（赤黒木は最大 2 log n）
- 検索重視 → AVL木、更新重視 → 赤黒木

---

## コード例（Python: BST の基本操作）

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

- **「BST は常に O(log n)」は誤り** — ランダムデータなら期待値 O(log n) だが、ソート済みデータで最悪 O(n)
- **AVL木と赤黒木は実装が複雑** — 面接・競技プログラミングでは `SortedList`（`sortedcontainers`）や言語標準の `TreeMap` を使う
- **B-Tree と BST は別物** — DBのインデックスは B-Tree（1ノードに複数キー）でディスクI/Oを最小化。BST は主にメモリ内データ構造
- **削除処理が最難関** — 子を2つ持つノードの削除は「中順後継ノード（右部分木の最小値）」で置き換える

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のインデックス**:  
  `CREATE INDEX` で作られるB-Treeインデックスの基礎がBST。`EXPLAIN ANALYZE` でインデックスを使った検索が O(log n) になっているか確認できる

- **範囲検索の効率化**:  
  BST の中順探索 = ソート順アクセスの原理で、`WHERE created_at BETWEEN ... AND ...` がインデックスで高速化される仕組みを理解できる

- **FastAPI でのソート済みコレクション管理**:  
  メモリ内でソート順を維持したい場合（例: リアルタイムランキング）、`sortedcontainers.SortedList` が赤黒木相当の O(log n) 挿入・検索を提供

```python
# pip install sortedcontainers
from sortedcontainers import SortedList

scores = SortedList(key=lambda x: -x[1])  # スコア降順
scores.add(("user_a", 95))
scores.add(("user_b", 87))
# 常にソート済みで O(log n) 挿入
```
