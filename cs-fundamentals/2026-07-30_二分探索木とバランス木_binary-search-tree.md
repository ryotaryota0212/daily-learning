# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、探索・挿入・削除を効率的に行える。  
ただし素のBSTは入力の偏りで偏った木になりO(n)に劣化する。  
AVL木・赤黒木は**自動的にバランスを保つ**ことでO(log n)を保証する。  
データベースのB-Tree、Java の`TreeMap`、Python の`sortedcontainers`など実務の根幹を支える仕組み。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは「左部分木 < 自分 < 右部分木」を満たす
- 探索: 比較しながら左右どちらかに進む。最悪O(n)（一直線になった場合）
- **偏りの原因**: 昇順データを挿入すると右のみにつながりリストと同じになる

```
正常な木         偏った木（昇順 1,2,3,4,5 挿入）
     4              1
    / \              \
   2   5              2
  / \                  \
 1   3                  3
                          \
                           4
                            \
                             5
```

### AVL木
- **バランス条件**: 各ノードで`|左の高さ - 右の高さ| ≤ 1`
- 挿入・削除後にバランス因子を計算し、違反があれば**回転**で修正
- **4種の回転**: LL回転、RR回転、LR回転、RL回転
- 読み取りが多いワークロードに向く（バランスが厳格でツリーが低い）

### 赤黒木
- ノードに「赤」か「黒」の色を付ける
- **5つの制約**:
  1. 全ノードは赤か黒
  2. ルートは黒
  3. 赤ノードの子は必ず黒（赤が連続しない）
  4. 全葉（NILノード）は黒
  5. 任意のノードからNILまでの経路上の黒ノード数が同じ（黒高さ一定）
- AVL木より**回転回数が少なく書き込みが速い**
- Linux カーネルのプロセス管理、C++の`std::map`、Java の`TreeMap`に採用

---

## 計算量・パフォーマンス特性

| 操作 | BST（最良/最悪） | AVL木 | 赤黒木 |
|------|----------------|-------|--------|
| 探索 | O(log n) / O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) / O(n) | O(log n) | O(log n) |
| 削除 | O(log n) / O(n) | O(log n) | O(log n) |
| 回転コスト | なし | 最大2回 | 最大3回 |
| 木の高さ | 不定 | ≤ 1.44 log n | ≤ 2 log n |

**ポイント**: AVL木は木が低く探索に有利、赤黒木は回転が少なく書き込みに有利。

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
            if val == node.val:
                return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: バランスが保証されないとO(n)になる
- **AVL木と赤黒木の使い分けを混同**: 探索重視→AVL、挿入削除重視→赤黒木
- **削除が一番複雑**: 削除対象に子が2つある場合、中順後継（右部分木の最小値）で置き換えが必要
- **Pythonにはheapqはあるがbalanced BSTがない**: `sortedcontainers.SortedList`（内部はBList）か自前実装が必要
- **B-Treeと混同しない**: データベースのB-Treeは多分木（複数のキーを1ノードに持つ）でBSTの拡張版

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

**Neon（PostgreSQL）**
- `CREATE INDEX` で作られるB-Treeインデックスは赤黒木の発展形（ページ分割対応）
- `EXPLAIN` で `Index Scan` が出るとO(log n)の木探索が走っている証拠
- 範囲クエリ（`WHERE id BETWEEN 100 AND 200`）はBSTの中順走査を活用

**FastAPI**
- ルーティングの内部はトライ木（Trie）が多いが、ミドルウェアの優先度管理でヒープ/順序集合が使われる
- レートリミットの実装で時刻順のイベント管理に`SortedList`が有効

**一般的な応用パターン**
```
・重複排除しつつ順序保持 → SortedList / TreeSet
・範囲内の要素を数える → セグメント木（BSTの応用）
・k番目の要素を探す   → 順序統計木（各ノードに部分木サイズを持つBST）
```

> **学習の核心**: 「なぜDBのインデックスはハッシュより範囲検索が速いか」→ BSTは中順走査で自然にソート済み、ハッシュは順序を持たないから。
