# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序性を持ち、挿入・検索・削除を効率化するデータ構造。
しかし素朴なBSTは偏りが生じてO(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用で使われる。
実務では直接実装することは稀だが、PostgreSQLのB-Treeインデックスやlibstdc++の`std::map`など、
内部実装を理解することで「なぜO(log n)が保証されるか」が分かる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは`左子 < 自分 < 右子`の不変条件を保つ
- 検索：根から比較しながら左右に降りる → O(h)（hは木の高さ）
- **問題点**：ソート済みデータを順番に挿入すると連結リスト状になりh = n → O(n)に劣化

```
挿入順 1,2,3,4,5 の場合：
1
 \
  2
   \
    3  ← 線形になる
```

### AVL木

- **高さバランス条件**：全ノードで `|左の高さ - 右の高さ| ≤ 1`
- 挿入・削除後に条件違反が起きたら「回転」で修正
- 回転の種類：LL回転、RR回転、LR回転、RL回転（1〜2回で修正完了）
- 高さが厳密に保たれるため検索が速い。挿入・削除は回転コストあり

### 赤黒木

- 各ノードに「赤 or 黒」の色情報を持つ
- **5つの条件**（赤黒性質）で高さをO(log n)に制約：
  1. 各ノードは赤か黒
  2. 根は黒
  3. 葉（nil）は黒
  4. 赤ノードの子は必ず黒
  5. 任意のノードから葉までの黒ノード数は等しい（黒高さ）
- AVL木より緩いバランス → 挿入・削除が速い（回転回数の上限が少ない）
- C++の`std::map`、JavaのTreeMap、Linuxのスケジューラで採用

---

## 計算量・パフォーマンス特性

| 操作 | 素朴BST（最悪） | AVL木 | 赤黒木 |
|------|---------------|-------|--------|
| 検索 | O(n) | O(log n) | O(log n) |
| 挿入 | O(n) | O(log n) | O(log n) |
| 削除 | O(n) | O(log n) | O(log n) |
| 回転コスト | - | O(log n)回 | O(1)回（最大）|

- **AVL木の強み**：高さが最も低い → 検索ヘビーな場合に有利
- **赤黒木の強み**：挿入・削除の回転が少ない → 書き込みが多い場合に有利
- 空間計算量：いずれもO(n)

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

- **「BSTはO(log n)」は平均の話**：最悪O(n)になる。平衡が保証されるのはAVL木・赤黒木のみ
- **AVL木は常に最良ではない**：検索より更新が多いワークロードでは赤黒木の方が速い
- **B-TreeはBSTではない**：DBのインデックスで使われるB-Treeはノードが複数キーを持つ多分木（後日別記事）
- **ハッシュテーブルとの使い分け**：順序が必要（範囲検索、ソート済み出力）→ BST系、順序不要で点検索のみ → ハッシュテーブル

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連：**

- **PostgreSQL（Neon）のインデックス**：`CREATE INDEX`で作られるB-Treeは赤黒木を多分木に拡張したもの。`WHERE id > 100 AND id < 200`のような範囲検索が速いのはBST系の順序性のおかげ
- **APIのレスポンスのソート**：Pythonの`sorted()`はTimSortでO(n log n)だが、挿入しながらソート順を維持したい場合は`sortedcontainers.SortedList`（内部はB-Tree的な実装）が有効
- **レートリミット・スケジューラ**：有効期限付きキーを時刻順に管理するには赤黒木ベースの順序付き集合が適する（FastAPIのバックグラウンドタスクのキュー管理）
