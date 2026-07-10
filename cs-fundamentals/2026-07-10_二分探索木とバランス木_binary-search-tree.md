# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という制約を持つ木構造で、検索・挿入・削除をO(log n)で行える。
しかし偏った入力でO(n)に劣化するため、自己平衡するAVL木・赤黒木が実用される。
DBのインデックス（B-Tree）、言語標準ライブラリのmap/set（赤黒木ベース）など至る所に使われる基礎構造。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは「値・左の子・右の子」を持つ
- 検索：目的値と比較して左右を再帰的に辿る
- 挿入：末端まで辿り空いている場所に追加
- **問題点**：昇順で挿入すると一直線（連結リスト相当）になりO(n)

```
挿入: 5, 3, 8, 1, 4
      5
     / \
    3   8
   / \
  1   4
```

### AVL木
- 各ノードで「左右の高さの差 ≤ 1」を常に保つ
- 違反時に「回転」(Rotation) で修正: 右回転・左回転・二重回転
- 検索は常にO(log n)保証。挿入・削除のオーバーヘッドあり

### 赤黒木
- 各ノードを赤/黒に塗り、以下の4規則を守る:
  1. ルートは黒
  2. 赤ノードの子は必ず黒
  3. 任意のノードから葉までの黒ノード数は等しい
  4. 葉（NIL）は黒
- AVL木より回転回数が少なく、挿入・削除が速い
- Pythonの`sortedcontainers`、C++の`std::map`、Javaの`TreeMap`が採用

---

## 計算量・パフォーマンス特性

| 操作       | BST平均 | BST最悪 | AVL木  | 赤黒木 |
|------------|---------|---------|--------|--------|
| 検索       | O(log n)| O(n)    | O(log n)| O(log n)|
| 挿入       | O(log n)| O(n)    | O(log n)| O(log n)|
| 削除       | O(log n)| O(n)    | O(log n)| O(log n)|
| 回転コスト | -       | -       | 最大2回 | 最大3回|

- **メモリ**: 各ノードにポインタ2本（＋高さ or 色）→ ハッシュテーブルより多い
- **キャッシュ効率**: ポインタで繋がるのでキャッシュミスが起きやすい（B-Treeが優れる理由）

---

## コード例（Python: BST 検索と挿入）

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

- **「BSTは常にO(log n)」は誤り**: ソート済みデータを挿入すると木が偏りO(n)
- **AVL木の方が常に速い？**: 検索が多い場合はAVL木有利、挿入・削除が多い場合は赤黒木有利
- **削除は挿入より複雑**: 削除では「後継ノード（右部分木の最小値）」で置き換える処理が必要
- **自己平衡木を一から実装しない**: 本番ではSortedContainersや標準ライブラリを使う

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連**

- **Neon（PostgreSQL）のインデックス**: BTreeインデックスは赤黒木の発展形のB-Tree。`EXPLAIN ANALYZE`でIndex Scanが出るのはこの仕組みが動いている
- **範囲検索**: ハッシュインデックスと違いBSTは`BETWEEN`や`ORDER BY`に強い。Neonでの`WHERE created_at BETWEEN ...`は必ずBTree列を使う
- **FastAPIでの順序付きデータ**: Pythonの`sortedcontainers.SortedList`（赤黒木ベース）でリアルタイムランキング等を実装できる
- **Cloud Runでのキャッシュ**: プロセス内で順序付きキャッシュが必要なら赤黒木ベースの構造が有効（インスタンスの再起動で消えるため補助的に使う）
