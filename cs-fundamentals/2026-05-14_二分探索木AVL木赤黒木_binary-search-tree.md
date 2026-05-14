# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序を維持するツリー構造で、探索・挿入・削除を効率的に行える。
しかし単純なBSTは入力次第でO(n)に劣化するため、自己平衡を行うAVL木・赤黒木が実用される。
Pythonの`sortedcontainers`、Java の`TreeMap`、DBのB-Treeインデックスなど、実務の随所に登場する。
「なぜDBインデックスにBSTではなくB-Treeが使われるか」を理解する土台になる。

---

## 仕組みの要点

### 二分探索木（BST）

- **探索**: ルートから「探す値 < 現在 → 左、大きければ → 右」を繰り返す
- **挿入**: 探索の要領で空きノードに追加
- **削除**: 3パターン（子なし / 子1つ / 子2つ）
  - 子2つの場合は「中順後継（右部分木の最小値）」で置き換える
- **問題点**: ソート済みデータを挿入すると一直線になり、深さがO(n)になる

```
挿入順: 3, 1, 5, 4, 6
      3
     / \
    1   5
       / \
      4   6

挿入順: 1, 2, 3, 4, 5（最悪ケース）
1
 \
  2
   \
    3  ← 深さ O(n)
```

### AVL木

- 各ノードで「左右部分木の高さの差（平衡因子）」を管理
- 平衡因子が ±2 になったら**回転操作**で修正（左回転・右回転・二重回転）
- 高さは常に O(log n) を保証
- 挿入/削除ごとに回転が発生するため、書き込み頻度が高い用途では赤黒木より遅いことがある

```
挿入後に不平衡 → 左回転で修正:

  1             2
   \           / \
    2    →    1   3
     \
      3
```

### 赤黒木

- 各ノードを赤か黒に色付け、以下の5条件を維持:
  1. ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒
  5. 任意のノードからNILまでの黒ノード数は同じ（黒高さ）
- 高さは最大 2×log(n+1) → O(log n) を保証
- AVL木より回転回数が少なく、**挿入・削除が高速**
- C++の`std::map`、LinuxカーネルのCFS（スケジューラ）で採用

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木     | 赤黒木    |
|----------|------------|------------|-----------|-----------|
| 探索     | O(log n)   | O(n)       | O(log n)  | O(log n)  |
| 挿入     | O(log n)   | O(n)       | O(log n)  | O(log n)  |
| 削除     | O(log n)   | O(n)       | O(log n)  | O(log n)  |
| 空間     | O(n)       | O(n)       | O(n)      | O(n)      |

- AVL木は高さが厳密に低い → **探索頻度が高い**場合に有利
- 赤黒木は回転が少ない → **挿入・削除が多い**場合に有利

---

## コード例（Python）

```python
class BSTNode:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, val):
            if node is None:
                return BSTNode(val)
            if val < node.val:
                node.left = _ins(node.left, val)
            elif val > node.val:
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

- **「BSTは常にO(log n)」は誤り** — 平衡が保たれないとO(n)に劣化
- **自前実装は避けるべき** — 平衡木の実装は複雑でバグが出やすい。ライブラリを使う
- **赤黒木 ≠ 最速** — 読み取り専用ならAVL木の方が探索は速い
- **インデックスにはB-Tree** — BSTは1ノード1要素でディスクI/Oが多い。DBはB-Treeで1ノードに複数要素を格納してI/Oを削減する

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neonの裏側（PostgreSQL）のインデックス**
  - `CREATE INDEX` はB-Tree（赤黒木の多分岐版）を使う
  - 範囲検索（`BETWEEN`、`>`, `<`）が効くのはツリー構造で順序が保たれるため
  - ハッシュインデックスでは範囲検索不可 → 使い分けの理解が必要

- **ソート済み集合が必要な場面**
  - APIのレスポンスでスコアや日時で順位付けするロジックに`sortedcontainers.SortedList`が使える
  - 内部は赤黒木ベースで挿入O(log n)、二分探索O(log n)

- **Cloud RunのCPUスケジューリング**
  - Linuxカーネルの完全公平スケジューラ（CFS）は赤黒木でタスクを管理
  - コンテナのvCPU割り当てがどう機能するかの理解に繋がる

```python
# FastAPIで使う例: ソート済み順位表の管理
from sortedcontainers import SortedList

ranking = SortedList(key=lambda x: -x["score"])
ranking.add({"user": "alice", "score": 95})
ranking.add({"user": "bob", "score": 87})
# 常にスコア降順で管理、挿入O(log n)
```
