# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は検索・挿入・削除をすべて同じ構造で行える汎用データ構造。ただし、偏ったデータを入れると最悪O(n)に劣化する。AVL木・赤黒木は自己平衡によりO(log n)を保証する。辞書型やセット、DBのインデックスなど「順序を保ちながら高速検索したい」場面で核心になる概念。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左 < 自分 < 右」の不変条件を満たす
- 検索: ルートから大小比較で半分ずつ絞り込む
- 挿入: 検索と同じ手順で末尾に追加
- **問題**: ソート済みデータを順番に挿入すると一直線のリストになりO(n)**

```
挿入順 1→2→3→4
1
 \
  2
   \
    3       ← 高さ=n、検索がO(n)に劣化
     \
      4
```

### AVL木

- 各ノードで「左右の高さの差 ≤ 1」を常に維持
- 挿入・削除後に回転操作（単純回転・二重回転）で再バランス
- 高さ ≤ 1.44 log₂(n) が数学的に保証される
- **読み取り頻度が高い場面に向く**（赤黒木より厳密に均衡）

### 赤黒木

- 各ノードに「赤/黒」の色情報を追加
- 4つのルール（根は黒、赤の連続禁止、すべての経路の黒ノード数が等しい等）で高さをO(log n)に保証
- 高さ ≤ 2 log₂(n+1) が保証（AVLより緩いが十分）
- **書き込み頻度が高い場面に向く**（回転回数が少なく挿入・削除が高速）
- 多くの言語標準ライブラリの実装に採用（C++ `std::map`、Java `TreeMap`、Linux kernel等）

---

## 計算量・パフォーマンス特性

| 操作 | BST平均 | BST最悪 | AVL木 | 赤黒木 |
|------|---------|---------|-------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

- AVL木の挿入時の最大回転数: **2回**
- 赤黒木の挿入時の最大回転数: **2回**（削除も最大3回）

---

## コード例（Python: 単純BST）

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
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTはいつもO(log n)」は誤り**。平均ケースの話であり、最悪はO(n)
- **AVLと赤黒木は「どちらが速いか」より「何に向くか」で選ぶ**。読み取り重視→AVL、書き込み重視→赤黒木
- **Pythonの`dict`と`set`はハッシュテーブル**であり二分探索木ではない。順序が必要なら`sortedcontainers.SortedDict`等を使う
- 赤黒木の「黒の高さが等しい」ルールは全パス（根→葉）について成り立つ必要がある点を忘れがち

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**:

- **Neon（PostgreSQL）のB-Tree索引**: 内部は赤黒木に近い考え方を持つB-Tree。`CREATE INDEX`の計算量がO(log n)である理由はここにある
- **範囲クエリ**: `WHERE created_at BETWEEN ...` がB-Treeインデックスで効率的なのは、木構造が順序を保持するから（ハッシュインデックスでは範囲検索不可）
- **FastAPIのルーティング**: フレームワーク内部でのルート検索もトライ木（木構造の応用）ベースのことが多い
- **ログのソート済み挿入**: アクセスログを時系列に保ちながら追記する場合、挿入順がほぼ昇順になるため素のBSTは劣化する。実際のDBはB-Treeで対処している
