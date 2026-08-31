# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、検索・挿入・削除をO(log n)で行える。しかし偏った挿入順ではO(n)に劣化する。AVL木と赤黒木は**自己平衡**することでこの最悪ケースを防ぐ。RDBのB-Treeインデックス、Pythonの`sortedcontainers`、Javaの`TreeMap`など実務で広く使われる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `値・左子・右子` を持つ
- 検索: 値が小さければ左、大きければ右に再帰的に降りる
- 挿入: 検索と同じ経路の末端に追加
- 削除: 子の数で3パターン（葉・1子・2子）。2子の場合は中順後継ノードで置換
- **問題点**: 1,2,3,4...と昇順に挿入すると完全な右片寄りリストになりO(n)

```
挿入順: 1,2,3,4,5 → 最悪ケース
1
 \
  2
   \
    3
     \
      4
```

### AVL木

- 各ノードが「左部分木の高さ - 右部分木の高さ = balance_factor」を持つ
- `balance_factor ∈ {-1, 0, 1}` を常に保持
- 挿入・削除後にbfが ±2 になったらrotation（回転）で修正
- 4種の回転: LL回転、RR回転、LR回転、RL回転
- **特徴**: 厳密なバランスで検索が速い。挿入・削除で回転が多い

### 赤黒木

- 各ノードが色（赤 or 黒）を持つ
- 5つの不変条件（RB properties）:
  1. 全ノードは赤か黒
  2. 根は黒
  3. 全葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意ノードからNILへの経路上の黒ノード数はすべて等しい（黒高さ）
- 高さの上限 ≤ 2 * log₂(n+1) → 疑似バランス
- **特徴**: AVL木より緩いバランスだが回転回数が少なく挿入・削除が高速

---

## 計算量・パフォーマンス特性

| 操作       | BST（平均） | BST（最悪） | AVL木    | 赤黒木   |
|-----------|-----------|-----------|---------|---------|
| 検索       | O(log n)  | O(n)      | O(log n)| O(log n)|
| 挿入       | O(log n)  | O(n)      | O(log n)| O(log n)|
| 削除       | O(log n)  | O(n)      | O(log n)| O(log n)|
| 空間       | O(n)      | O(n)      | O(n)    | O(n)    |

- AVL木は定数係数が小さく**読み取り多用**シナリオに有利
- 赤黒木は**書き込み多用**シナリオに有利（Linux カーネル、C++ STL map/set が採用）

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

    def insert(self, val, node=None, is_root=True):
        if is_root:
            node = self.root
        if node is None:
            n = Node(val)
            if self.root is None:
                self.root = n
            return n
        if val < node.val:
            node.left = self.insert(val, node.left, False)
        elif val > node.val:
            node.right = self.insert(val, node.right, False)
        return node

    def search(self, val, node=None, start=True):
        if start:
            node = self.root
        if node is None:
            return False
        if val == node.val:
            return True
        if val < node.val:
            return self.search(val, node.left, False)
        return self.search(val, node.right, False)
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ランダム挿入なら平均O(log n)だが最悪O(n)。ソート済みデータの一括挿入は要注意
- **AVL木と赤黒木を「同じもの」と思わない** — バランスの厳密さが異なり、定数係数のトレードオフがある
- **回転は値のコピーではなくポインタ操作** — 既存ノードへの参照が壊れないよう注意
- **削除の「中順後継」** — 右部分木の最小ノード（一番左を辿る）を使う実装が一般的
- **黒高さの定義** — NIL（番兵）ノードを黒と数えるかどうかで実装が変わる

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のB-Treeインデックス**: BSTの考え方を拡張したB-Treeがベース。`EXPLAIN ANALYZE` でインデックスが使われているか確認する際、「高さが低いほど検索が速い」という直感が直接役立つ
- **範囲クエリの最適化**: BSTの中順走査（inorder）が昇順になる性質 → `BETWEEN`や`ORDER BY`がインデックスを効率的に使う理由を理解できる
- **ソート済みデータの一括INSERT**: マイグレーションやシードデータをIDの昇順で挿入するとB-Treeの再バランスコストが高い → ランダムUUIDの方が有利な場面がある
- **Cloud Run（アプリ内キャッシュ）**: `sortedcontainers.SortedList`（内部的に赤黒木相当）を使うとキャッシュのTTL管理や範囲削除をO(log n)で実現できる
