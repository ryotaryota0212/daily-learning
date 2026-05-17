# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）はデータを順序付きで管理し、O(log n)の検索・挿入・削除を実現する基本構造。
しかし素朴なBSTは最悪O(n)に劣化するため、自己平衡木（AVL木・赤黒木）が生まれた。
データベースインデックス（B-Tree）やプログラミング言語の標準ライブラリ（Python `sortedcontainers`、Java `TreeMap`）の基盤となる重要概念。

---

## 仕組みの要点

### 二分探索木（BST）の基本

- 各ノードは「左の子 < 自分 < 右の子」を満たす
- 中順巡回（左→自分→右）でソート済みリストが得られる
- **問題点**: 昇順データを挿入すると木が一直線になりO(n)に退化

```
通常のBST（バランスが良い）    偏ったBST（最悪ケース）
        5                        1
       / \                        \
      3   7                        2
     / \ / \                        \
    2  4 6  8                        3  ← 連結リストと同じ
```

### AVL木

- 各ノードの左右部分木の高さの差（バランス因子）を **-1, 0, 1** に保つ
- 挿入・削除後に**4種類の回転**（左回転・右回転・左右・右左）でバランスを修正
- 回転は局所的な操作でO(1)。全体の再構築は不要
- 赤黒木より**検索が速い**（より厳密にバランスが保たれる）
- 欠点: 挿入・削除のたびに多くの回転が必要になる場合がある

### 赤黒木

- 各ノードを赤か黒に色付けし、以下の規則を守る：
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数は等しい（黒高さ一定）
- AVL木より**緩い**バランス条件 → 挿入・削除の回転回数が少ない
- 実務でよく使われる（Linux カーネルのスケジューラ、多くの標準ライブラリ）

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- AVL木の高さ ≤ 1.44 × log₂(n+2)
- 赤黒木の高さ ≤ 2 × log₂(n+1)
- **検索頻度が高い** → AVL木が有利
- **挿入・削除が頻繁** → 赤黒木が有利

---

## コード例（Python: 素朴なBST）

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

- **「BSTは常にO(log n)」は誤り** — ソート済みデータを挿入するとO(n)に劣化する
- **AVL木と赤黒木は用途が違う** — 「どちらが優れているか」ではなく、アクセスパターン次第
- **Pythonの`dict`や`set`はハッシュテーブル** — BSTではないので順序保証なし（挿入順は保証）
- **Python標準ライブラリにBSTはない** — `sortedcontainers.SortedList` やheapqで代替
- **削除がもっとも複雑** — 子が2つあるノードの削除は中順後継ノード（右部分木の最小値）で置換する

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neonのインデックス（B-Tree）**: PostgreSQLのデフォルトインデックスはB-Treeで、内部的に赤黒木の考え方を拡張したもの。`CREATE INDEX`の裏側を理解する基礎になる
- **範囲クエリの効率**: `WHERE created_at BETWEEN ...` がハッシュインデックスではなくB-Treeインデックスで高速な理由がわかる
- **ORMのorder_by**: SQLAlchemy経由のソートはDBがインデックスをBSTとして辿る操作
- **FastAPIでのソート済み応答**: インメモリでソート済みコレクションが必要なら `sortedcontainers.SortedList` を検討（挿入のたびに`sort()`するより効率的）

### 面接・設計での活用

- 「なぜこのDBのクエリが遅いか」を説明する際のインデックス設計の議論に直結
- キャッシュの優先度管理（LRU + 順序付きアクセス）でBSTベースの構造が登場する
