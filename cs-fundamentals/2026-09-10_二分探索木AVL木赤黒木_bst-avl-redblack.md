# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は検索・挿入・削除をO(log n)で行える木構造だが、**偏りが生じるとO(n)に劣化する**。
AVL木と赤黒木はそれぞれ異なるバランス戦略でO(log n)を保証する自己平衡木。
データベースのインデックス（B-Tree）、言語標準ライブラリのMapやSet（Pythonの`sortedcontainers`、C++の`std::map`）の内部実装として直接効いてくる知識。

---

## 仕組みの要点

### 二分探索木（BST）

- **条件**: 各ノードで「左の子 < 自分 < 右の子」
- **検索**: ルートから比較して左右を選ぶだけ。深さ分の比較で完了
- **弱点**: 昇順でデータを挿入すると右にしか伸びない → 線形リストと同等になる

```
挿入順: 1, 2, 3, 4, 5
     1
      \
       2
        \
         3  ← 深さ = n-1, 検索O(n)に劣化
```

### AVL木

- **バランス条件**: 各ノードの左右部分木の高さの差（バランス因子）が **-1, 0, +1** の3値のみ許容
- **回転操作**: 挿入・削除後にバランスが崩れたら「回転」で修正
  - 右回転・左回転（単純回転）
  - 左右回転・右左回転（二重回転）
- **特徴**: 厳密なバランスなのでBSTより高さが低い → **検索が速い**
- **代償**: 挿入・削除時の回転頻度が高く更新コスト大

### 赤黒木

- **バランス条件（5つのルール）**:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードからすべての葉までの「黒ノード数」が等しい
- **特徴**: AVL木より緩いバランス → 高さは最大 2log(n+1)
- **代償**: 検索はAVL木よりわずかに遅いが、**挿入・削除の回転が少なく更新が速い**
- **使われている場所**: Linux kernel のスケジューラ、Java `TreeMap`、C++ `std::map`

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

- AVL木は赤黒木より高さが低い → **検索ヘビーなら AVL木が有利**
- 赤黒木は挿入・削除時の再バランスが少ない → **更新ヘビーなら赤黒木が有利**

---

## コード例（Python：BST の基本操作）

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
            if val == node.val:
                return True
            node = node.left if val < node.val else node.right
        return False
```

実用には `sortedcontainers.SortedList` を使うことで赤黒木相当の性能を得られる。

---

## よくある誤解・落とし穴

- **「BSTはO(log n)」は保証でなく期待値**: ランダムなデータなら平均O(log n)だが、ソート済みデータでO(n)に崩壊する
- **AVL木 = 常に最速ではない**: 書き込み多用途では回転コストが積み重なり赤黒木より遅くなる場合がある
- **赤黒木のルールを暗記しようとする**: ルールは「任意の根から葉までのパスの長さが最大2倍以内」という結果を生む仕組みと理解する方が定着する
- **バランス木 ≠ B-Tree**: B-Tree はノードが複数のキーを持つ多分木（ディスクI/O最適化）。赤黒木とは別物

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のインデックス**: PostgreSQL は B-Tree インデックスを使用。B-Tree は赤黒木の発展形で、ページ単位でバランスを保つ多分木。`EXPLAIN ANALYZE` でインデックス検索の深さを意識するとクエリチューニングの勘が養われる
- **Python の `sortedcontainers`**: FastAPI のリクエスト処理中にソート済みデータを頻繁に検索・更新する場合（例：ランキング、タイムラインのカーソル管理）にO(log n)を保証する `SortedList` が有効
- **Cloud Run のオートスケール**: コンテナ起動数の管理に内部的に優先度キュー（ヒープ）や平衡木が使われる。自分が直接触ることはないが「なぜスケールが速いか」の背景理解につながる
- **キャッシュの有効期限管理**: TTL付きキャッシュを自前で実装する際、期限切れエントリを効率的に見つけるために赤黒木やヒープが使われるパターンがある
