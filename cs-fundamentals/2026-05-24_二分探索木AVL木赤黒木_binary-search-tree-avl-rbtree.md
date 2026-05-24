# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

「ソート済みデータへの高速アクセス」を実現する木構造の中核。ハッシュテーブルが O(1) でも順序を保持できないのに対し、BST系は**順序を保ちながら O(log n) で検索・挿入・削除**を実現する。PostgreSQLのB-Treeインデックス（B-Treeは多分岐版BST）、Pythonの `sortedcontainers`、Javaの `TreeMap` など実装例多数。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左 < 自分 < 右」の性質を持つ
- 探索：根から比較しながら左右に降りる → 最悪 O(n)（偏った木）
- **問題点**：挿入順によっては線形リストに退化する

```
挿入順: 1→2→3→4→5 の場合（最悪ケース）
1
 \
  2
   \
    3  ← 高さ n、探索 O(n)
```

### AVL木（高さ平衡二分探索木）

- 任意のノードで「左右の高さ差 ≤ 1」を保証（**バランス係数**）
- 挿入・削除後に**回転操作**（左回転・右回転・二重回転）でリバランス
- 高さが常に O(log n) → 探索・挿入・削除すべて O(log n) 保証
- **トレードオフ**：回転頻度が高く、更新コストがやや大きい

```
右回転の例（左が重い場合）:
    C          B
   /    →    /  \
  B          A    C
 /
A
```

### 赤黒木（Red-Black Tree）

- 各ノードに「赤 or 黒」の色を付与。以下の規則で**緩やかなバランス**を維持:
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. すべての葉（nil）までの黒ノード数が同じ
- 高さ ≤ 2 × log(n+1) → O(log n) 保証だが AVL より少し高くなり得る
- **AVL との比較**：回転回数が少ない → **挿入・削除が多いユースケースで有利**
- C++ `std::map`、Linux カーネルのスケジューラ等で採用

---

## 計算量まとめ

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 回転数/挿入 | — | — | ≤ 2 | ≤ 2 |
| 回転数/削除 | — | — | O(log n) | ≤ 3 |

**AVL vs 赤黒木の選択基準**
- 読み取り多 → AVL（高さが低く検索有利）
- 書き込み多 → 赤黒木（リバランスコストが小さい）

---

## コード例（Python: BST の挿入と中順探索）

```python
class Node:
    def __init__(self, val):
        self.val, self.left, self.right = val, None, None

class BST:
    def __init__(self): self.root = None

    def insert(self, val):
        def _ins(node, v):
            if not node: return Node(v)
            if v < node.val: node.left = _ins(node.left, v)
            elif v > node.val: node.right = _ins(node.right, v)
            return node
        self.root = _ins(self.root, val)

    def inorder(self):
        res = []
        def _dfs(n):
            if n: _dfs(n.left); res.append(n.val); _dfs(n.right)
        _dfs(self.root); return res

bst = BST()
for v in [5, 3, 7, 1, 4]: bst.insert(v)
print(bst.inorder())  # [1, 3, 4, 5, 7]
```

---

## よくある誤解・落とし穴

- **「BSTは常に O(log n)」は誤り**：ランダムデータなら平均 O(log n) だが、ソート済みデータを挿入すると O(n) に退化する
- **AVL木は「完全平衡」ではない**：高さ差 1 は許容される（完全平衡は不要なコスト）
- **赤黒木はAVLより常に速いわけではない**：参照が多い場合はAVLの方が木が低いため有利
- **Pythonの `dict` はBSTではない**：ハッシュテーブル。順序保証が必要なら `sortedcontainers.SortedDict` を使う

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **Neon（PostgreSQL）のインデックス**: 実体は B-Tree（BSTの多分岐版）。`CREATE INDEX` で暗黙に作成され、`WHERE id = ?` や `ORDER BY` を O(log n) に最適化
- **範囲クエリの効率化**: `BETWEEN`, `>`, `<` はハッシュインデックスでは非対応 → B-Treeインデックスが必須。「なぜ B-Tree か」を理解していると `EXPLAIN ANALYZE` の結果が読める
- **FastAPI でのソート済みデータ管理**: インメモリで順序付きコレクションが必要な場合、`sortedcontainers.SortedList` が赤黒木ベースで O(log n) を提供
- **Cloud Run のオートスケール時のクエリ最適化**: 同時接続増加時のボトルネック特定に、インデックス（B-Tree）の理解が直結する
