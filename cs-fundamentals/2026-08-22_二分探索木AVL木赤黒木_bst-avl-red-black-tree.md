# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 根 < 右」の性質を持つ木構造で、辞書・集合・ソート済みデータの保持に使われる。
しかし素朴なBSTは偏りが生じると線形時間に劣化する。これを解決するのが**自己平衡二分探索木**（AVL木・赤黒木）。
実務では `dict/set`（Python）の内部実装やデータベースのインデックスに直結する概念。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `key`、`left`、`right` を持つ
- 検索：根から「小なら左、大なら右」を再帰的にたどる
- 挿入・削除も同様のパスをたどった末に操作
- **問題**：昇順データを順に挿入すると一本鎖になり `O(n)` に劣化

```
挿入: 5, 3, 7, 1, 4

    5
   / \
  3   7
 / \
1   4
→ 高さ = 3 (バランス良好)

挿入: 1, 2, 3, 4, 5

1
 \
  2
   \
    3  ← 線形リストと同等 O(n)
```

### AVL木

- **条件**：任意のノードで `|左の高さ - 右の高さ| ≦ 1`
- 挿入・削除後に違反が生じたら**回転**で修正
  - 右回転（Left-Left ケース）
  - 左回転（Right-Right ケース）
  - 左右回転・右左回転（LR/RL ケース）
- 高さは常に `O(log n)` を保証
- **特徴**：検索が多い用途に向く（赤黒木より高さが低い）

### 赤黒木（Red-Black Tree）

- 各ノードに **赤 or 黒** の色属性を付加
- **5つのルール**
  1. 全ノードは赤か黒
  2. 根は黒
  3. 葉（NILノード）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの**黒ノード数は等しい**（黒高さ）
- 挿入・削除後に色の変更＋回転で修正（最大2回の回転）
- 高さ `≦ 2 log(n+1)`
- **特徴**：AVL木より挿入・削除が速い（回転回数が少ない）

---

## 計算量・パフォーマンス特性

| 操作 | 素朴BST | AVL木 | 赤黒木 |
|------|---------|-------|--------|
| 検索 | O(n) 最悪 | O(log n) | O(log n) |
| 挿入 | O(n) 最悪 | O(log n) | O(log n) |
| 削除 | O(n) 最悪 | O(log n) | O(log n) |
| 回転数（最悪） | - | O(log n) | O(1)〜O(log n) |
| メモリ | 最小 | +高さ情報 | +色ビット |

- AVL木は高さが `≈ 1.44 log n`、赤黒木は `≈ 2 log n` → 検索はAVL有利
- 挿入・削除の多いワークロードでは赤黒木が有利
- Python の `sortedcontainers.SortedList` は実装上スキップリストを使う（赤黒木ではない）

---

## コード例（Python: 素朴BST）

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, key):
        def _ins(node, key):
            if not node:
                return Node(key)
            if key < node.key:
                node.left = _ins(node.left, key)
            elif key > node.key:
                node.right = _ins(node.right, key)
            return node
        self.root = _ins(self.root, key)

    def search(self, key):
        node = self.root
        while node:
            if key == node.key:
                return True
            node = node.left if key < node.key else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常に O(log n)」は誤り**。平衡を保つ仕組みがなければ最悪 O(n)
- AVL木と赤黒木の選択基準：「検索多→AVL」「書き込み多→赤黒木」がざっくりした指針だが、実測で判断するのが正確
- 赤黒木の削除は最も複雑なケースが多い（6パターンほどのケース分岐）。手実装より `sortedcontainers` 等のライブラリ利用推奨
- 「回転＝木の再構築」ではない。ポインタ付け替えだけで O(1) で完了する局所操作

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

| 場面 | 関連 |
|------|------|
| **Neon (PostgreSQL) のB-Treeインデックス** | B-Treeは赤黒木の変種（多分岐版）。`EXPLAIN ANALYZE` で Index Scan が出るとき、内部はこの仕組みで動いている |
| **範囲クエリ（`BETWEEN`, `ORDER BY`）** | BSTの中順走査でソート済み列挙が O(n) で可能。これがB-Treeインデックスが範囲検索に強い理由 |
| **FastAPIのルーティング** | フレームワーク内部でルートをツリー管理。原理を知ることでパスパラメータのマッチング順を理解できる |
| **キャッシュの有効期限管理** | 優先度付きキューや平衡木でTTL順管理。Redis の Sorted Set はスキップリスト + ハッシュテーブルで同等機能を実現 |

**実践ヒント**：Neonで `EXPLAIN (ANALYZE, BUFFERS) SELECT ...` を実行し、`Index Scan using xxx` の行を確認すると、今日学んだ木構造がクエリを高速化している様子を直接観察できる。
