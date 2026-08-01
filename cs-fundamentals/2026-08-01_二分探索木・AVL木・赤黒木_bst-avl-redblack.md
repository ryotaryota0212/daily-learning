# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、探索・挿入・削除をO(log n)で行える基盤データ構造。ただし偏った入力でO(n)に劣化するため、自己平衡木（AVL木・赤黒木）が実用では使われる。PythonのsortedContainers、PostgreSQLのB-Treeインデックスなど、実務の至る所でこの原理が動いている。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `left < node <= right` を満たす
- 探索：根から比較しながら左右どちらかを下る
- 挿入：探索と同じ経路を辿り末尾に追加
- 削除：後継ノード（右部分木の最小値）と置換して処理
- **問題点**：昇順データを挿入すると完全に偏り、線形リストになる

```
挿入順: 1,2,3,4,5 → 右に一直線（高さ=n）
        1
         \
          2
           \
            3  ← O(n)の探索になる
```

### AVL木

- 各ノードで `|左の高さ - 右の高さ| <= 1` を常に維持
- 挿入・削除後に「回転」で再平衡化（右回転・左回転・二重回転）
- 高さは常にO(log n)を保証
- **特徴**：BSTより厳格なバランス → 探索は最速、挿入/削除はやや重い

```
右回転の例:
    z              y
   /      →      / \
  y             x   z
 /
x
```

### 赤黒木

- 各ノードを「赤」か「黒」に色付け。以下の5条件を守る：
  1. 各ノードは赤か黒
  2. 根は黒
  3. 葉（nil）は黒
  4. 赤ノードの子は必ず黒
  5. 任意のノードから葉までの黒ノード数は同じ
- AVLより緩いバランス（高さ ≤ 2log(n+1)）→ 回転回数が少ない
- **特徴**：挿入/削除が速い → C++ `std::map`、Java `TreeMap`、Linux カーネルのプロセス管理で採用

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木  | 赤黒木 |
|----------|------------|------------|--------|--------|
| 探索     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間     | O(n)       | O(n)       | O(n)   | O(n)   |
| 回転回数 | -          | -          | O(log n) | O(1)（挿入）|

- AVL木は回転が多い分、探索ヘビーなユースケースで有利
- 赤黒木は書き込みヘビーなユースケースで有利

---

## コード例（Python：BSTの基本操作）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, val):
            if not node:
                return Node(val)
            if val < node.val:
                node.left = _ins(node.left, val)
            else:
                node.right = _ins(node.right, val)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**：ランダムな入力なら期待値O(log n)だが最悪O(n)。ソート済みデータで完全に壊れる
- **AVL木 = 常に最速、ではない**：回転コストがあるため書き込みが多い場合は赤黒木の方が速い
- **Pythonの `dict` や `set` は木でなくハッシュテーブル**：順序付き操作（範囲検索、最小値取得）が必要なら `sortedcontainers.SortedList` や `sorteddict` を使う
- **削除は実装が最も複雑**：後継ノードの処理や平衡調整で多くのバグが生まれる

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **NeonのB-Treeインデックス**：PostgreSQLのインデックスはB-Tree（BSTの多分木版）が基本。`CREATE INDEX` で暗黙に使われる。範囲検索（`WHERE created_at BETWEEN ...`）や `ORDER BY` の最適化に直結
- **クエリ最適化**：`EXPLAIN ANALYZE` でIndex Scanが出れば木構造での探索が行われている証拠
- **ソート済みデータの一括挿入は危険**：CSVインポートなどでIDが昇順だと内部ツリーが偏ることがある（PostgreSQLは自動でリバランスするが、挿入速度への影響は出る）
- **Cloud Run上のメモリ内キャッシュ**：優先順位付きキューやソート済み集合が必要な場面では、Pythonの `heapq`（ヒープ）か `sortedcontainers` が候補になる。木構造の原理を知っていると使い分けが迷わない
