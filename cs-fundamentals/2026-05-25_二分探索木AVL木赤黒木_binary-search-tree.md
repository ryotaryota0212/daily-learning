# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造。この制約により探索・挿入・削除が効率的に行える。しかし素朴なBSTは入力順によって偏り（最悪O(n)）が生じるため、**自己平衡木**（AVL木・赤黒木）が実用で使われる。PostgreSQLのB-Treeインデックスや言語標準ライブラリ（Python `sortedcontainers`、Java `TreeMap`）の内部で活用されており、「ソート済み集合への高速な挿入・削除・範囲検索」が必要な場面で直接効いてくる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノード: `左の子 < 自分 < 右の子` を満たす
- 探索: ルートから大小比較を繰り返し目標に到達
- 挿入: 探索と同じ経路で空きノードに追加
- **問題点**: 昇順に挿入すると右に伸びた線形リスト（O(n)）になる

```
挿入順: 3, 1, 4, 1, 5, 9  →  バランス良好
挿入順: 1, 2, 3, 4, 5     →  右に偏った鎖状ツリー（最悪ケース）
```

### AVL木（Adelson-Velsky and Landis, 1962）

- **平衡条件**: 全ノードで `|左の高さ - 右の高さ| ≤ 1`
- 挿入・削除後にこの条件が崩れたら**回転（Rotation）**で修復
- 4パターンの回転: LL, RR, LR, RL
- 赤黒木より厳密なバランスのため**探索が高速**（読み取り多用途向き）
- 代わりに挿入・削除時の回転が多め

### 赤黒木（Red-Black Tree, 1972）

- 各ノードに赤/黒の色を付け、4つのルールで弱い平衡を保証:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 全パスの黒ノード数は等しい（黒高さが一定）
  4. NILノードは黒とみなす
- AVL木より緩い平衡 → **挿入・削除の回転が少なく高速**
- Linux カーネル、C++ `std::map`、Java `TreeMap` で採用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- AVL木の木の高さ: 最大 `1.44 × log₂(n)`
- 赤黒木の木の高さ: 最大 `2 × log₂(n + 1)`（AVLより若干高い）

---

## コード例（Python: 簡易BST探索）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

def insert(root, val):
    if root is None:
        return Node(val)
    if val < root.val:
        root.left = insert(root.left, val)
    else:
        root.right = insert(root.right, val)
    return root

def search(root, val):
    if root is None or root.val == val:
        return root
    return search(root.left, val) if val < root.val else search(root.right, val)
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: 平衡が保証されていないBSTは最悪O(n)
- **「AVL木が常に最速」は誤り**: 書き込みが多い場合は赤黒木の方が実用的
- **削除は挿入より複雑**: 中順後継ノード（右部分木の最小値）で置き換える必要がある
- **Python標準にBSTはない**: `sortedcontainers.SortedList` がO(log n)操作を提供（実際はB-Tree系）

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連:**

- **Neon（PostgreSQL）のインデックス**: B-Tree（BSTの多分木拡張）が`CREATE INDEX`のデフォルト。範囲クエリ（`BETWEEN`, `ORDER BY`）が高速になる仕組みはここにある
- **API設計**: ランキング・リーダーボード機能で「特定スコア以上のユーザーを取得」する場合、DBインデックスがBSTベースの範囲探索をしていると理解することで、適切なインデックス設計ができる
- **in-memoryキャッシュ**: `sortedcontainers.SortedDict` で有効期限順の優先処理（期限切れのキャッシュを効率的に削除）などに使える
