# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という不変条件を持つ木構造で、検索・挿入・削除が平均 O(log n) で行える。ただし最悪ケースで O(n) に劣化する問題がある。AVL木・赤黒木はそれを自動的に解消する**自己平衡二分探索木**で、Python の `sortedcontainers`、Java の `TreeMap`、PostgreSQL のインデックスなど幅広い場面で使われる。

---

## 仕組みの要点

### 二分探索木（BST）

- **不変条件**: `left.val < node.val < right.val`（全ノードで成立）
- **検索**: ルートから値を比較し左右へ降りる
- **問題点**: 昇順挿入などで木が線形（偏った木）になり O(n) に劣化

```
挿入: 1 → 2 → 3 → 4 （最悪ケース）
1
 \
  2
   \
    3
     \
      4   ← 連結リストと同じ形
```

### AVL木

- **高さバランス条件**: 各ノードで `|左の高さ - 右の高さ| ≤ 1`
- **ローテーション**: 挿入・削除後にバランスが崩れたら回転で修正
  - 左回転・右回転・左右回転・右左回転の4種類
- **高さ**: 常に O(log n) を保証
- **特徴**: 赤黒木より厳密なバランス → 検索が速い、挿入・削除のコストはやや高い

```
右回転（例）:
    5              3
   /     →        / \
  3              2   5
 / \                /
2   4              4
```

### 赤黒木

- **4つの条件**:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 赤ノードの子は必ず黒（赤が連続しない）
  4. どのパスも同じ黒ノード数（black-height が一定）
- **AVL木との違い**: 完全なバランスより「ほぼバランス」を許容 → 挿入・削除が速い
- **使われる場所**: Linux カーネルのスケジューラ、C++ `std::map`、Java `TreeMap`

---

## 計算量

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| ローテーション回数 | — | — | O(log n) | O(1) |

**赤黒木がよく使われる理由**: 挿入・削除時のローテーションが最大 O(1) 回で済む（AVL 木は O(log n) 回必要なことがある）ため、書き込みが多いワークロードに有利。

---

## コード例（Python：BST の基本操作）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, v):
            if not node:
                return Node(v)
            if v < node.val:
                node.left = _ins(node.left, v)
            elif v > node.val:
                node.right = _ins(node.right, v)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val:
                return True
            node = node.left if val < node.val else node.right
        return False
```

> **実用**: Python では `from sortedcontainers import SortedList` を使うと赤黒木相当の O(log n) 操作が手軽に利用できる。

---

## よくある誤解・落とし穴

- **「BST は常に速い」** → ソート済みデータを挿入すると O(n) に劣化する。入力がランダムでない場合は AVL 木・赤黒木を使うこと
- **「ローテーションは複雑だから実装不要」** → 仕組みを理解していないと、ライブラリの挙動（なぜ O(log n) なのか）がブラックボックスになる
- **「ヒープと同じ」** → ヒープは最大・最小取得に特化、BST 系は任意の検索・順序走査が得意。用途が異なる
- **「インデックス = ハッシュテーブル」** → DB の B-Tree インデックスは範囲検索が得意（BST 系の考え方）、ハッシュインデックスは等値検索のみ

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run スタック）

- **Neon (PostgreSQL) のインデックス**: `CREATE INDEX` で作られる B-Tree は赤黒木の考え方を拡張したもの。`WHERE id BETWEEN 100 AND 200` が速いのはこれが理由
- **FastAPI での範囲フィルタ設計**: ソート済みの順序が必要な場合（タイムライン、ランキング）は DB インデックスに任せ、インメモリで SortedList を使うと効率的
- **Cloud Run のオートスケール**: リクエストキューの管理にヒープ・優先度付き構造が使われる（実装は非公開だが原理は同じ）
- **範囲クエリの最適化**: BST 系構造のおかげで `ORDER BY created_at LIMIT 20` が O(log n + k) で完了することを理解しておくと、クエリチューニングの判断が速くなる
