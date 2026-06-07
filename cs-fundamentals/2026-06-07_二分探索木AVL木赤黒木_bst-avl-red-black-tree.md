# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序性を保つ木構造で、検索・挿入・削除をO(log n)で行える。  
ただし偏り（skew）が生じるとO(n)に劣化するため、自己平衡木（AVL木・赤黒木）が実用では使われる。  
PythonのsortedcontainersやDB内部のインデックス構造にも同じ原理が応用されており、計算量の保証を理解することが設計判断に直結する。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左部分木 ≤ 自分 < 右部分木」を満たす
- 検索: rootから大小比較しながら下る → 平均O(log n)
- **問題点**: 昇順に挿入すると一直線になり、高さがn → O(n)に劣化

```
挿入順 1,2,3,4 → 偏り発生
1
 \
  2
   \
    3
     \
      4   ← 実質的に連結リスト
```

### AVL木

- **高さのバランス条件**: 各ノードで「左右の高さの差（balance factor）≦ 1」を強制
- 挿入・削除後に違反が起きたら**回転（rotation）**で修正
- 四種類の回転: 左回転 / 右回転 / 左右回転 / 右左回転
- 赤黒木より厳密なバランス → 検索が若干速い、挿入・削除のコストが高い

### 赤黒木

- 各ノードに「赤 or 黒」の色を付け、以下の性質を維持:
  1. rootは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. どのパスもNILまでの黒ノード数が等しい（黒高さ一定）
- 高さの上限は `2 * log(n+1)` → バランスが保証される
- AVL木より緩いバランス → 回転頻度が少なく、挿入・削除が高速
- **実用**: Linux カーネルのスケジューラ、C++ `std::map`、Java `TreeMap`

---

## 計算量

| 操作       | BST（平均） | BST（最悪） | AVL木    | 赤黒木   |
|------------|-------------|-------------|----------|----------|
| 検索       | O(log n)    | O(n)        | O(log n) | O(log n) |
| 挿入       | O(log n)    | O(n)        | O(log n) | O(log n) |
| 削除       | O(log n)    | O(n)        | O(log n) | O(log n) |
| 空間       | O(n)        | O(n)        | O(n)     | O(n)     |

- AVL木の回転: 最悪O(log n)回
- 赤黒木の回転: 挿入・削除とも定数回（O(1)）で収まる

---

## コード例（Python: BSTの基本構造）

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

実用上はPythonの `sortedcontainers.SortedList`（内部はリストの分割統治）や  
`heapq`（ヒープ）で代替することが多い。

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ランダムデータなら平均O(log n)だが、ソート済みデータでO(n)劣化
- **AVL木の方が常に速いわけではない**: 書き込みが多いユースケースでは赤黒木の方が高速
- **Pythonに標準の平衡木はない**: `dict`/`set`はハッシュテーブル。順序が必要なら `sortedcontainers` を使う
- **削除操作は挿入より複雑**: 後継ノード（in-order successor）を使った置き換えが必要

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**

- **NeonのB-Treeインデックス**: PostgreSQLのインデックスはB-Tree（BSTを多分木に拡張）。`WHERE id = X` の速度保証の根拠が赤黒木の考え方と同じ
- **範囲クエリの効率**: `WHERE created_at BETWEEN A AND B` が速いのはB-Treeが順序を保持するから。ハッシュインデックスでは不可
- **FastAPIのルーティング**: 内部的に木構造でルートを管理。パスパラメータの衝突解決に順序木の考えが使われる
- **ログ調査**: 「なぜこのクエリが遅いか」を `EXPLAIN ANALYZE` で見るとき、インデックスのseqスキャンとindex scanの違いを木の高さで直感的に理解できる
