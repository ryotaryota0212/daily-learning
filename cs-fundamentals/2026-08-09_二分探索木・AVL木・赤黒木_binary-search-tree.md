# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という順序制約を持つ木構造で、探索・挿入・削除を効率的に行える。
しかし素朴なBSTは入力順によって偏り（最悪O(n)）が生じる。AVL木と赤黒木はその問題を自動的な**回転操作**で解決する平衡二分探索木。
実務では直接実装することは稀だが、データベースインデックス（B-Tree）・言語標準ライブラリの`SortedMap`等の内部で使われており、「なぜO(log n)が保証されるか」を理解することでパフォーマンス予測精度が上がる。

---

## 仕組みの要点

### 二分探索木（BST）

- **探索**: ルートから比較を繰り返し目的ノードへ降りる
- **挿入**: 探索と同じ経路をたどり、空き葉に追加する
- **削除**: 子の数によって3パターン（子なし・子1つ・子2つ）で処理
- **問題点**: ソート済み列を順番に挿入すると右に偏った線形リストになり、高さO(n)

```
挿入順: 5, 3, 7, 1, 4    挿入順: 1, 2, 3, 4, 5 (最悪)
    5                       1
   / \                       \
  3   7                       2
 / \                           \
1   4                           3  ← 線形リスト (O(n))
```

### AVL木（高さ平衡）

- 全ノードで「左右の高さの差（平衡因子） ≤ 1」を維持
- 挿入・削除後に平衡が崩れたら**回転**（左回転・右回転・二重回転）で修正
- 高さは常にO(log n)が保証される
- **特徴**: BSTより厳密に平衡 → 探索が高速。挿入・削除はやや遅い（回転が多い）

```
右回転の例（Zが不平衡）:
    Z              Y
   / \    →       / \
  Y   C          X   Z
 / \                / \
X   B              B   C
```

### 赤黒木（色による近似平衡）

- 各ノードに赤か黒の色を付与し、以下の5条件を維持:
  1. ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの黒ノード数は同じ（黒高さ一定）
- 条件から高さ ≤ 2log(n+1) が証明できる → O(log n)保証
- **特徴**: AVLより緩い平衡 → 回転が少なく挿入・削除が速い。探索はわずかに遅い

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転数（最悪） | - | - | O(log n) | O(1) |

- **AVL木** → 探索頻度が高いシステム（読み取り多）に有利
- **赤黒木** → 挿入・削除が多いシステム（書き込み多）に有利。Linux kernel、Java `TreeMap`、C++ `std::map`で採用

---

## コード例（Python: BST探索と挿入）

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
            if node is None:
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

- **「BSTは常にO(log n)」は誤り**: ランダム挿入なら平均O(log n)だが、ソート済みデータでは必ずO(n)に劣化する
- **AVL木 = 赤黒木より優秀ではない**: 探索重視ならAVL、挿入削除重視なら赤黒木と使い分ける
- **平衡木の実装は複雑**: 実務では標準ライブラリの`SortedContainers`（Python）や`TreeMap`（Java）を使う。自前実装は回転・色修正のバグが多い
- **木の高さとノード数の関係**: 高さh → ノード数は最大2^(h+1)−1、最小h+1。高さO(log n)と言えるのは平衡が保たれているときだけ

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連:**

- **Neon（PostgreSQL）のB-Treeインデックス**: B-Treeは二分探索木を多分木に拡張したもの。`CREATE INDEX`で作成されるのがこれ。BSTの原理を理解すると「なぜ等価検索・範囲検索はインデックスを使えてLIKE '%foo'は使えないか」が直感的にわかる
- **FastAPIのルーティング**: パスパラメータの照合に内部でトライ木（BST系）を使うフレームワークが多い
- **Cloud Runのオートスケール判断**: リクエスト数のソート済み管理にヒープ（BST系）が使われる
- **アプリ実装**: Pythonで範囲クエリ・順序付きセットが必要なとき → `sortedcontainers.SortedList`（内部がB-Tree系）を使うと平衡木の恩恵をそのまま得られる

```python
from sortedcontainers import SortedList
sl = SortedList([5, 3, 7, 1])
sl.add(4)
print(sl[2])          # 4  (O(log n))
print(sl.irange(3,5)) # [3, 4, 5]  (範囲クエリ O(log n + k))
```
