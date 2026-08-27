# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序を保つ木構造で、探索・挿入・削除を O(log n) で行う土台となる。ただし入力順次第でO(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用では使われる。PostgreSQLのB-Tree、Pythonの`sortedcontainers`、Java の`TreeMap`など、順序付きデータが必要な場面で直接活きる知識。

---

## 仕組みの要点

### 二分探索木（BST）の基本

- **構造**: 各ノードが `left`, `right`, `value` を持つ
- **順序制約**: `left.value < node.value < right.value`（全サブツリーで再帰的に成立）
- **中順走査（in-order）** で昇順ソート済みリストが得られる
- **問題点**: `[1, 2, 3, 4, 5]` を順に挿入→ 右に伸びる線形リスト → 探索 O(n)

```
挿入順: 3, 1, 5, 2, 4 → バランス良い
       3
      / \
     1   5
      \ /
      2 4

挿入順: 1, 2, 3, 4, 5 → 片寄り（最悪ケース）
1
 \
  2
   \
    3
```

### AVL木（高さ平衡木）

- **高さバランス条件**: 各ノードで `|height(left) - height(right)| ≤ 1`
- **回転操作** で挿入・削除後に再平衡: 左回転・右回転・左右回転・右左回転の4種
- **高さ**: 常に O(log n) を保証
- **特徴**: BSTより厳密な平衡 → 探索速度高い、挿入・削除のコストやや高い

### 赤黒木（Red-Black Tree）

5つの条件でゆるい平衡を保つ:
1. 各ノードは赤か黒
2. 根は黒
3. 葉（NIL）は黒
4. 赤ノードの子は必ず黒（赤が連続しない）
5. 根から全葉までの「黒ノード数」が等しい（黒高さ一致）

- 「黒高さ一致」により高さは最大 2·log(n+1)
- AVL木より**平衡はゆるい**が、回転頻度が低く挿入・削除が速い
- **Linux カーネル**・C++ STL `map`・Java `TreeMap` で採用

---

## 計算量

| 操作 | BST平均 | BST最悪 | AVL木 | 赤黒木 |
|------|---------|---------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

- AVL木: 挿入で最大1回、削除で O(log n) 回の回転
- 赤黒木: 挿入・削除で最大3回の回転（定数）→ 償却コストが低い

---

## コード例（Python）

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

実用では `sortedcontainers.SortedList`（内部はB木）や `heapq` で代替可能。

---

## よくある誤解・落とし穴

- **「BSTは常に速い」**: ソート済みデータを順に挿入すると線形劣化。必ず平衡木を使う
- **「AVL木が最善」**: 探索が圧倒的に多い読み取り専用なら有利だが、書き込み多数なら赤黒木の方が実用的
- **「自前実装が必要」**: 実務では言語標準ライブラリに任せる。Pythonには標準BSTがない（`sortedcontainers`がデファクト）
- **削除が一番複雑**: 「後継ノード（右部分木の最小値）で置き換え」を忘れがち

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連**

- **Neon（PostgreSQL）**: `CREATE INDEX` はB-Tree（赤黒木の変形）。`WHERE id BETWEEN 100 AND 200` などの**範囲クエリ**はインデックスが中順走査で効率化
- **ソート済みキャッシュ**: FastAPI でレスポンスを価格順・日時順で返す際、`SortedList` を使えば毎回ソートせず O(log n) 挿入で維持可能
- **実行計画の読み方**: `EXPLAIN ANALYZE` で `Index Scan` が出ればB-Treeが機能している証拠。`Seq Scan` が出たらインデックス欠如を疑う
