# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「検索・挿入・削除をO(log n)で行う」データ構造の基盤。
ただし素朴なBSTは偏りが生じてO(n)に劣化する。AVL木・赤黒木はその問題を自動均衡で解決する。
実務では直接実装することはほぼないが、DBインデックス（B-Tree）やソートされたMap実装の根幹にあり、「なぜO(log n)か」を理解することがパフォーマンス分析の土台になる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左の子 < 自分 < 右の子」を満たす
- 中順走査（in-order traversal）で昇順ソート済みリストが得られる
- **問題点**: 昇順データを挿入し続けると右に偏った線形リストになり、高さがO(n)に

```
挿入順: 1, 2, 3, 4, 5
結果:
1
 \
  2
   \
    3  ← 線形リストと同じ。検索がO(n)に劣化
     \
      4
       \
        5
```

### AVL木

- **不変条件**: 任意のノードで「左右の部分木の高さの差 ≤ 1」
- 挿入・削除後に違反を検出 → **回転（rotation）** で均衡を回復
- 回転の種類: 右回転・左回転・左右回転・右左回転の4種
- 高さが常にO(log n)に保たれる → 操作もO(log n)保証

**回転のイメージ（右回転）**:
```
    z              y
   /      →      /  \
  y             x    z
 /
x
```

### 赤黒木（Red-Black Tree）

- 各ノードに「赤 or 黒」の色属性を付加
- **5つの不変条件**:
  1. 各ノードは赤か黒
  2. 根は黒
  3. 葉（NILノード）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの黒ノード数は等しい（黒高さ一定）
- AVL木より**緩い均衡条件** → 回転数が少なく挿入・削除が高速
- 高さ上限は `2 * log(n+1)` → O(log n)保証

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 最大/最小 | O(log n) | O(n)    | O(log n) | O(log n) |

**AVL vs 赤黒木**:
- AVL木: より厳密な均衡 → 検索が若干高速。挿入・削除の回転コストが大きい
- 赤黒木: 実用上の均衡 → 挿入・削除が高速。多くの言語標準ライブラリで採用
  - Python: `sortedcontainers`（実装はB木系）
  - Java: `TreeMap`, `TreeSet`（赤黒木）
  - C++ STL: `std::map`, `std::set`（赤黒木）

---

## コード例

BSTの基本操作（Python）:

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

- **「BSTはO(log n)」は平均の話**: ランダムデータなら成立するが、ソート済みデータでは必ずO(n)になる
- **AVL木と赤黒木の選択**: 検索が圧倒的に多い → AVL木。挿入・削除が多い → 赤黒木
- **自前実装の罠**: 回転ロジックのバグは非常に発見しにくい。実務では標準ライブラリを使い切ることが重要
- **B-TreeとBSTは別物**: DBインデックスで使われるB-Treeはノードに複数のキーを持ち、ディスクI/O最小化のために設計されている

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**:

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX`はB-Tree（BSTの多分木拡張）をデフォルトで使用。`WHERE id = ?`や`ORDER BY created_at`の効率化は赤黒木系の原理に基づく
- **範囲クエリ**: `BETWEEN`や`>`の効率的な実行はB-Treeの中順走査がベース。インデックスを張るかどうかの判断基準として計算量を理解することが重要
- **FastAPIのルーティング**: フレームワーク内部でトライ木やハッシュマップが使われているが、パスパラメータの照合にツリー構造の概念が関係
- **ログの時系列分析**: ソート済みデータへのアクセスパターンを理解することで、インデックス設計の判断（例: `(user_id, created_at)`の複合インデックス）に直結する
