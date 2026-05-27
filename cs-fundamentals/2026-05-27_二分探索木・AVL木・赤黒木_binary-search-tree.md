# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序を保つ木構造で、探索・挿入・削除をO(log n)で行える。
しかし偏りが生じるとO(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用される。
DBのインデックスや標準ライブラリの辞書型など、あらゆる場所で内部的に使われている。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左サブツリー < ノード値 < 右サブツリー」を満たす
- 探索：根から比較しながら左右に下る
- **問題**：昇順挿入すると一直線（最悪O(n)）になる

```
挿入順: 1→2→3→4 の場合
1
 \
  2
   \
    3
     \
      4  ← 連結リストと同じ状態
```

### AVL木

- 各ノードで「左右の高さの差（バランス因子）≤ 1」を保証
- 挿入・削除後に**回転操作**でバランスを回復
- 4パターンの回転：LL・RR・LR・RL

```
右回転（RR回転）の例:
    C          B
   /    →    /  \
  B          A    C
 /
A
```

- **長所**：高さが厳密にO(log n)、探索が速い
- **短所**：回転頻度が多く、挿入・削除コストが高め

### 赤黒木

- 各ノードを赤か黒に色付けし、5つの規則を維持
  1. 各ノードは赤か黒
  2. 根は黒
  3. 赤ノードの子は黒（赤が連続しない）
  4. すべての葉（NIL）は黒
  5. 任意のノードからNILまでの黒ノード数は同じ
- **高さの保証**：最大で2·log(n+1)、AVL木よりやや緩い
- **長所**：再バランス操作が少なく、挿入・削除が速い

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木   | 赤黒木  |
|----------|-------------|-------------|---------|---------|
| 探索     | O(log n)    | O(n)        | O(log n)| O(log n)|
| 挿入     | O(log n)    | O(n)        | O(log n)| O(log n)|
| 削除     | O(log n)    | O(n)        | O(log n)| O(log n)|

- **AVL木**：探索重視（回転多め、高さ最小）
- **赤黒木**：書き込み重視（Linux CFS、C++ `map`、Java `TreeMap` で採用）

---

## コード例（Python：BST探索）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

def insert(root, val):
    if not root:
        return Node(val)
    if val < root.val:
        root.left = insert(root.left, val)
    else:
        root.right = insert(root.right, val)
    return root

def search(root, val):
    if not root or root.val == val:
        return root
    if val < root.val:
        return search(root.left, val)
    return search(root.right, val)
```

---

## よくある誤解・落とし穴

- **「BSTなら常にO(log n)」は誤り**：ランダム挿入なら期待値O(log n)だが、ソート済みデータでO(n)に劣化
- **AVL木 > 赤黒木とは限らない**：挿入・削除が多いワークロードでは赤黒木が有利
- **削除は挿入より複雑**：BSTの削除は後継者（右部分木の最小値）または前任者で置換する処理が必要
- **Pythonの`sortedcontainers.SortedList`**は内部的にBSTではなくリストの配列を使用（実装を確認すること）

---

## 実務での使いどころ

- **FastAPI + Neon（PostgreSQL）スタック**
  - PostgreSQLのB-Treeインデックスは赤黒木の考え方を拡張したもの（ディスクI/O最適化版）
  - `WHERE id > 100 ORDER BY id` のような範囲クエリが速いのはBSTの順序性のおかげ
  - インデックスの仕組みを理解すると、どのカラムにインデックスを張るべきか判断できる

- **Cloud Run（APIサーバー）**
  - リクエストのルーティング処理や優先度付きキューの実装にヒープ/木構造が使われる
  - Pythonで範囲検索が必要なら `sortedcontainers` が実用的（BST相当の操作をO(log n)で提供）
