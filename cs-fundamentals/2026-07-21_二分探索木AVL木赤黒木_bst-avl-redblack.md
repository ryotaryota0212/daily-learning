# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の性質を持つ木構造で、検索・挿入・削除をO(log n)で行える。  
ただしデータが偏ると最悪O(n)に劣化するため、自己平衡機能を持つAVL木・赤黒木が実用される。  
PostgreSQLのB-Treeインデックス、Pythonの`sortedcontainers`、Java の`TreeMap`など、実装の根底にある仕組み。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードが「左サブツリー < 自ノード < 右サブツリー」を満たす
- 中順探索（左→親→右）で昇順ソート済みリストが得られる
- 昇順データを挿入するとリストと同じ状態になり高さがO(n)になる（最悪ケース）

### AVL木
- 各ノードで「左右サブツリーの高さ差 ≤ 1」を保証（バランス係数）
- 挿入・削除後に違反が起きたら**回転（Rotation）**で修正する
  - 右回転・左回転・左右回転・右左回転の4パターン
- BSTより厳格なバランスのため検索が高速だが、挿入・削除のコストが高め

### 赤黒木
- 各ノードを「赤」か「黒」に着色し、5つの性質を維持する
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの黒ノード数は等しい（黒高さ）
- AVL木より緩いバランス（高さ ≤ 2 * log₂(n+1)）
- 挿入・削除時の再着色と回転が少なく、更新が多いケースで有利

---

## 計算量

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|-------------|-------------|-------|--------|
| 検索 | O(log n)    | O(n)        | O(log n) | O(log n) |
| 挿入 | O(log n)    | O(n)        | O(log n) | O(log n) |
| 削除 | O(log n)    | O(n)        | O(log n) | O(log n) |
| 回転回数/挿入 | -      | -           | ≤2回  | ≤2回 |
| 回転回数/削除 | -      | -           | O(log n)回 | ≤3回 |

**選択指針**
- 検索が圧倒的に多い → AVL木（より厳格なバランス）
- 挿入・削除が頻繁 → 赤黒木（更新コストが低い）
- 実用上ほとんどの言語標準ライブラリは赤黒木を採用

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

- **「BSTは常にO(log n)」は誤り** — ソート済みデータを順番に挿入すると高さがO(n)になる
- **AVL木は必ずしも赤黒木より遅くない** — 読み取り専用ワークロードではAVL木が優秀
- **赤黒木の着色ルールを暗記しようとする** — 本質は「黒高さの均一性がバランスを保証する」こと
- **削除の複雑さを軽視** — 削除は挿入より実装が複雑（後継ノードの選び方が関わる）
- **ハッシュテーブルとの混同** — BSTは順序付き操作（範囲検索・ソート）が得意；ハッシュは等値検索のみ

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタック との関連**

- **Neon（PostgreSQL）のインデックス**: B-Treeインデックスは赤黒木を多分木に拡張したもの。`ORDER BY`や範囲条件（`BETWEEN`, `>=`）が速い理由はこの木構造にある
- **FastAPIのルーティング**: パスのプレフィックスマッチングに木構造が使われることがある
- **ソート済みデータの範囲取得**: `SELECT * FROM logs WHERE created_at BETWEEN ? AND ?` が速いのはB-Treeが左右の境界をO(log n)で特定できるから
- **インメモリキャッシュの実装**: 有効期限付きキャッシュのTTL管理にヒープや赤黒木が使われる（例：`sortedcontainers.SortedDict`）

**確認コマンド**
```sql
-- Neonでインデックスの種類を確認
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'your_table';
-- EXPLAIN ANALYZEで実行計画の木構造を確認
EXPLAIN ANALYZE SELECT * FROM logs WHERE created_at > '2026-01-01';
```
