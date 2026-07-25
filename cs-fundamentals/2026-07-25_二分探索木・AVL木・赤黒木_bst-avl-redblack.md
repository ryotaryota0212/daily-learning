# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 自分 < 右の子」という性質で効率的な探索を実現する木構造。しかし偏りが生じると O(n) に劣化するため、自己平衡木（AVL木・赤黒木）が実用で使われる。RDBのインデックス（B-Tree）やLinuxスケジューラにも赤黒木が採用されるCSの核心。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノード: `値`・`左の子`（< 自分）・`右の子`（> 自分）
- 探索: 値と比較しながら左右を再帰的に辿る
- **問題点**: 昇順データを挿入すると一直線になり、深さ = n → O(n) 劣化

### AVL木

- **バランス因子** = 左右の高さの差（-1, 0, 1 のみ許容）
- 因子が ±2 になったら**回転**で再平衡（右/左/LR/RL の4種）
- 高さ常に O(log n)、**探索が多いケースで有利**

### 赤黒木

- ノードに赤/黒の色を付加し5性質を維持:
  1. 各ノードは赤か黒
  2. 根は黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒
  5. 任意ノードから葉までの黒ノード数が同じ（黒高さ一定）
- 高さ上限: 2 × log₂(n+1) → O(log n) 保証
- AVL木より平衡が緩い分、**回転が少なく挿入/削除が速い**
- 採用例: Linux CFS スケジューラ、Java `TreeMap`、C++ `std::map`

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n)  | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)  | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)  | O(n)       | O(log n) | O(log n) |

選び方: **探索 >> 更新** → AVL木、**更新が頻繁** → 赤黒木

---

## コード例（Python: BST の挿入・探索）

```python
class Node:
    def __init__(self, val):
        self.val, self.left, self.right = val, None, None

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
        n = self.root
        while n:
            if val == n.val: return True
            n = n.left if val < n.val else n.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常に O(log n)」は誤り**: ソート済みデータで O(n) 劣化。実用では自己平衡木を使う
- **重複の扱い**: BSTは重複なし実装が多い。許す場合は `≤` で左右どちらかに統一する
- **Python標準ライブラリにBSTはない**: `sortedcontainers.SortedList` を使う（内部はB-Arrayベース）

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX` はデフォルトで **B-Tree** を使用。赤黒木の平衡概念が直接つながる

```sql
CREATE INDEX idx_users_email ON users(email);
EXPLAIN SELECT * FROM users WHERE email = 'x@example.com';
-- Index Scan → B-Tree経由、Seq Scan → インデックス未使用のサイン
```

- **クエリチューニング**: 選択率の低いカラムへのインデックス付与判断に計算量の理解が必要
- **Cloud Run でのメモリキャッシュ**: `SortedDict` で範囲クエリを実現する際、O(log n) の挿入コストを意識した設計ができる
