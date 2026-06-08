# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という性質を持つ木構造で、探索・挿入・削除をO(log n)で行える。しかし単純なBSTは偏りが生じると最悪O(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用的に使われる。PostgreSQLのB-Tree、Javaの`TreeMap`、Linuxカーネルのスケジューラなど、あらゆる場所で使われる基本データ構造。

---

## 仕組みの要点

### 二分探索木（BST）の基本

```
       8
      / \
     3   10
    / \    \
   1   6    14
```

- **探索**: ルートから「目標 < 現在 → 左、目標 > 現在 → 右」と辿る
- **挿入**: 探索して空のリーフ位置に挿入
- **削除**: 子が0→そのまま削除、子が1→置き換え、子が2→右部分木の最小値と交換
- **問題**: 昇順データを挿入すると一直線になりO(n)に劣化

### AVL木

- **平衡条件**: 各ノードの左右の高さの差（平衡因子）が `-1, 0, 1` のいずれか
- **回転操作**: 挿入・削除後に平衡因子がずれたら回転で修正
  - 左回転・右回転・左右回転・右左回転の4パターン
- **特徴**: 非常に厳格な平衡 → 探索が速い、挿入・削除でのリバランスコストがやや高い

### 赤黒木（Red-Black Tree）

- **5つの規則**:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 赤ノードの子は黒（赤が連続しない）
  4. NILノード（番兵）は黒
  5. どのノードからNILまでの黒ノード数は同じ（黒高さ一定）
- **AVL木との違い**: やや緩い平衡 → 挿入・削除が高速、探索はAVL木より若干遅い
- **実用例**: Linux CFS scheduler（`struct rb_root`）、C++ `std::map`、Java `TreeMap`

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

- AVL木の高さ ≤ 1.44 × log₂(n+2)
- 赤黒木の高さ ≤ 2 × log₂(n+1)（AVL木より最大2倍高くなりうる）

---

## コード例（Python: BSTの探索と挿入）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def insert(self, root, val):
        if not root:
            return Node(val)
        if val < root.val:
            root.left = self.insert(root.left, val)
        else:
            root.right = self.insert(root.right, val)
        return root

    def search(self, root, val):
        if not root or root.val == val:
            return root
        if val < root.val:
            return self.search(root.left, val)
        return self.search(root.right, val)
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ランダム挿入なら平均O(log n)だが、ソート済みデータで偏る
- **AVL木 vs 赤黒木の選択**: 読み取りが多い→AVL木、書き込みが多い→赤黒木が有利
- **自前実装は避ける**: 回転ロジックはバグを生みやすい。言語標準ライブラリを使うべき
- **BSTとB-Treeを混同しない**: DBのB-Treeは複数キーを持つ多分木で別物

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連:**

- **PostgreSQLのインデックス**: NeonはPostgreSQLベースなので、`CREATE INDEX`はB-Tree（赤黒木の発展版）を使う。範囲クエリ（`WHERE id BETWEEN 100 AND 200`）が効く理由はこの構造のおかげ
- **ソート済み結果の取得**: BSTの中順走査（in-order traversal）はソート済みリストをO(n)で生成できる。インデックスを使ったORDER BYがそれに相当
- **キャッシュの有効期限管理**: 有効期限をキーとする赤黒木で最も早く期限切れになるエントリをO(log n)で取り出す設計はCloudフロントのCDNキャッシュ実装で使われる
- **FastAPI**: Pythonの`sortedcontainers.SortedList`（内部はBST系）を使えばリアルタイムランキングAPIを効率的に実装できる
