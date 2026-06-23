# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という順序制約を持つ木構造。探索・挿入・削除がO(log n)で動作するが、バランスが崩れるとO(n)に劣化する。AVL木と赤黒木はこの劣化を防ぐ「自己平衡木」。実務ではDBのインデックス（B-Tree）、言語標準ライブラリのMap/Set（多くが赤黒木）に使われており、計算量保証を理解することでデータ構造選択の判断精度が上がる。

---

## 仕組みの要点

### 二分探索木（BST）

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \
      4   7
```

- 探索: ルートから「左か右か」を比較しながら降りる → O(h)（h=木の高さ）
- 挿入: 探索して末端に追加
- 削除: 子が0個/1個は単純。子が2個の場合は「右部分木の最小値」で置換
- **問題**: ソート済みデータを挿入するとh=nの線形リストになる

### AVL木

- 各ノードで「左右の高さの差（バランス因子）≤1」を保証
- 挿入・削除後に違反が生じたら**回転**で修正（4パターン）
  - LL回転: 左-左の偏り → 右回転1回
  - RR回転: 右-右の偏り → 左回転1回
  - LR回転: 左-右の偏り → 左回転 + 右回転
  - RL回転: 右-左の偏り → 右回転 + 左回転
- 高さが常にO(log n)に保たれる → 探索が高速
- **欠点**: 頻繁な挿入・削除時に回転コストが高い

### 赤黒木

- 各ノードを「赤」か「黒」に色付け、5つの不変条件で平衡を保つ
  1. ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. どのノードからも葉までの経路上の黒ノード数は同じ
- AVL木より緩い平衡基準（高さ ≤ 2×log n）
- 挿入・削除時の回転回数が最大2回と少ない → 書き込み多用途で有利

---

## 計算量・パフォーマンス特性

| 操作   | BST（最悪） | AVL木    | 赤黒木   |
|--------|-------------|----------|----------|
| 探索   | O(n)        | O(log n) | O(log n) |
| 挿入   | O(n)        | O(log n) | O(log n) |
| 削除   | O(n)        | O(log n) | O(log n) |
| 回転数 | —           | O(log n) | O(1)     |

- AVL木は高さが厳密に低い → 探索はわずかに速い
- 赤黒木は挿入・削除時の回転が少ない → 書き込み多い場合に有利
- Pythonの`sortedcontainers.SortedList`はB-Tree系、C++の`std::map`は赤黒木

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

- **「BSTは常にO(log n)」は誤り**: バランス保証がないBSTは挿入順次第でO(n)になる。本番で使うなら自己平衡木を選ぶか、ライブラリ実装を使う
- **AVL木と赤黒木の使い分けを間違える**: 読み取り中心→AVL木（高さが低い）、書き込み多い→赤黒木（回転少ない）
- **削除の複雑さを軽視する**: 子2個のノード削除は「後継者（successor）」に置換後に後継者を削除する2ステップ。回転まで含めるとバグの温床になりやすい
- **ライブラリで十分な場面で自前実装する**: 競プロ以外では`sortedcontainers`等を使えばよい

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連**

- **Neon（PostgreSQL）のインデックス**: B-Tree（BSTを多分木に拡張）が基本インデックス。`CREATE INDEX`時に`USING btree`（デフォルト）。範囲検索（`BETWEEN`、`>`）に強く、ハッシュインデックスは完全一致のみ
- **クエリ最適化**: `EXPLAIN ANALYZE`でindex scanとseq scanを比較する際、B-Treeの特性（ソート済み順次アクセスが高速）を理解していると読み方がわかる
- **FastAPI でのソート済みコレクション管理**: 優先度付きキューや順位管理に`sortedcontainers.SortedList`を使うと内部でバランス木が機能する
- **Cloud Run のリクエスト管理**: 接続プールやリクエストルーティングの内部実装でB-Treeや赤黒木が使われているケースが多い（アプリ層では意識不要だが、パフォーマンス問題の根拠として理解しておくと有益）
