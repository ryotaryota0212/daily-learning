# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の不変条件を持つ木構造で、ソート済みデータへの高速アクセスを実現する。
素朴なBSTは偏りが生じるとO(n)に劣化するため、AVL木・赤黒木といった**自己平衡BST**が実用的に使われる。
PostgreSQLのB-Treeインデックスや言語標準ライブラリ（PythonのsortedcontainersなどC++の`std::map`）の実装に直結する知識。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `key`, `left`, `right`, `parent` を持つ
- 探索：左右どちらに進むか決めながら根から葉へ降りる
- 挿入：探索と同じ経路の末尾に葉として追加
- 削除：子の数に応じて3パターン（子なし/1つ/2つ）
- **問題点**：ソート済みデータを順番に挿入すると一方向リストになりO(n)

### バランスの概念

```
バランス良い:         バランス悪い（偏り）:
      4                   1
     / \                   \
    2   6                   2
   / \ / \                   \
  1  3 5  7                   3  ← 線形リスト
```

### AVL木

- 各ノードで `|height(left) - height(right)| <= 1` を保持
- 違反時に**回転（rotation）**で修正（左回転・右回転・二重回転）
- 挿入後は最大O(log n)回の回転が必要
- 探索が非常に高速な一方、挿入・削除のコストがやや高い

### 赤黒木（Red-Black Tree）

- 各ノードに色（赤/黒）を付加し、以下の5条件を維持：
  1. すべてのノードは赤か黒
  2. 根は黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒
  5. どの葉からも根までの黒ノード数が等しい（黒高さ）
- 回転 + 色の付け替えでバランス回復、AVL木より回転回数が少ない
- Linuxカーネルのスケジューラ、Java の `TreeMap`、C++の `std::map` に採用

---

## 計算量・パフォーマンス特性

| 操作     | 素のBST（平均） | 素のBST（最悪） | AVL木     | 赤黒木    |
|--------|------------|------------|---------|---------|
| 探索     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 回転回数   | —          | —          | O(log n) | O(1) 償却 |
| バランス厳密度 | —         | —          | 高い      | やや緩い   |

- AVL木は赤黒木より高さが低く探索が速い（特に読み取り多数の場合）
- 赤黒木は挿入・削除時の回転が少なく書き込み多数の場合に向く

---

## コード例（Python: 素のBSTと中順探索）

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, key):
        def _insert(node, key):
            if node is None:
                return Node(key)
            if key < node.key:
                node.left = _insert(node.left, key)
            elif key > node.key:
                node.right = _insert(node.right, key)
            return node
        self.root = _insert(self.root, key)

    def inorder(self):
        result = []
        def _inorder(node):
            if node:
                _inorder(node.left)
                result.append(node.key)
                _inorder(node.right)
        _inorder(self.root)
        return result  # ソート済みリストになる
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ランダムデータなら期待値O(log n)だが、最悪O(n)。本番ではAVL/赤黒木を使うこと
- **AVL木と赤黒木の使い分けを知らない** — 読み取り多数ならAVL、書き込み多数なら赤黒木が有利
- **自己実装の罠** — 回転ロジックはバグを埋め込みやすい。実用では言語標準の`SortedList`や`SortedDict`を使う
- **ハッシュテーブルとの混同** — BSTは順序を保持し範囲検索が得意、ハッシュは点探索が速いが順序なし

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

**Neon（PostgreSQL）のインデックス**
- `CREATE INDEX ON table(column)` は内部でB-Treeインデックス（BSTの多分木版）を使用
- `WHERE column BETWEEN a AND b` のような範囲クエリが速いのはBSTの「中順探索」が活きるから
- `EXPLAIN ANALYZE` でインデックスが使われているか確認できる

**FastAPIのルーティング**
- URLルーティングにはトライ木（Trie）が使われる場合があるが、内部的にはBSTと同じ「木による絞り込み」の考え方

**実践チェックリスト**
- Neonでの範囲検索カラムには必ずインデックスを貼る
- 複合インデックスのカラム順は「絞り込み力が高い順」に（BSTの深さを最小化する考え方と同じ）
- 挿入・更新が多いテーブルでは不要なインデックスを増やさない（B-Tree再バランスのコスト）
