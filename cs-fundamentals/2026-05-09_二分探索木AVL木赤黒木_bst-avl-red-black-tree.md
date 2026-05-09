# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という性質を持つ木構造。
データベースのインデックス、標準ライブラリのMap/Set、範囲検索など、「順序付きデータの高速な検索・挿入・削除」が必要な場面で中核を担う。
ただし素朴なBSTは入力順によって偏りが生じ最悪O(n)に退化するため、AVL木・赤黒木などの**自己平衡木**が実務で使われる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `左の部分木 < 自分 < 右の部分木` を保つ
- 検索: ルートから大小比較しながら降りる → 平均O(log n)
- **問題**: ソート済みデータを順に挿入すると一本鎖になり O(n) に退化

```
挿入順: 1, 2, 3, 4, 5
    1
     \
      2
       \
        3  ← 連結リストと同じ最悪ケース
```

### AVL木

- 各ノードで「左右の部分木の高さ差（バランス因子）≦ 1」を常に維持
- 挿入・削除後に**回転（Rotation）**で再バランス
  - 単回転（LL回転, RR回転）、二重回転（LR回転, RL回転）の4種類
- 高さが厳密に保たれるため検索は赤黒木より高速（低い木）
- 挿入・削除で回転が多く発生するため書き込み頻度が高い場合はやや遅い

### 赤黒木

- 各ノードを「赤」か「黒」に塗り分け、以下の性質を維持:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. どの葉からルートまでの「黒ノード数」が等しい（黒高さが一定）
- これにより木の高さ ≦ 2 × log(n+1) が保証される
- AVL木より平衡条件が緩い → 回転回数が少ない → 挿入・削除が速い
- **C++のstd::map、JavaのTreeMap、Linux カーネルのスケジューラ**などで採用

### 回転のイメージ（LL回転）

```
挿入前（バランス崩壊）    右回転後
      z                    y
     /                   /   \
    y          →         x     z
   /
  x
```

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n) |

- AVL木は高さが ≦ 1.44 × log(n) に抑えられ、赤黒木より木が低い
- 赤黒木は回転の最大回数が少ない（挿入:2回, 削除:3回）
- **読み取り多用** → AVL木有利。**書き込み多用** → 赤黒木有利

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
            if node is None:
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

- **「BSTは常にO(log n)」は誤り** — ランダムデータなら平均O(log n)だが、ソート済み入力で最悪O(n)
- **自己平衡木は魔法ではない** — 平衡維持のオーバーヘッドがあるため、小データセットではソート済み配列の二分探索の方が速い場合も
- **削除が難しい** — BSTの削除は「後継ノード（中順後継）への置換」が必要で実装ミスが起きやすい
- **重複値の扱い** — 標準仕様は重複を禁止。許可する場合はルール（左以下 or 右以下）を明示して統一する

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **NeonのB-Tree インデックス**はBSTの発展形（B-Treeは多分岐、ディスクI/O最適化）。BSTの原理を理解するとインデックス設計の直感が鋭くなる
- **範囲クエリ** (`WHERE created_at BETWEEN ...`) がインデックスで効く理由は、木構造の中順走査が順序を保つから
- **PythonのSortedContainers**（`SortedList`, `SortedDict`）は赤黒木相当の実装。FastAPIで「上位K件の取得」や「タイムラインのウィンドウ集計」に使える
- Cloud Run上でのインメモリキャッシュにソート済み構造が必要な場面では、`heapq`（ヒープ）かSortedContainersを選択肢に入れる

---

*作成日: 2026-05-09 | シリーズ: CS基礎 #3*
