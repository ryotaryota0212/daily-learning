# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序不変条件により、O(log n)での検索・挿入・削除を実現するデータ構造。
しかし偏った挿入順（例：昇順）でO(n)に退化する。これを解決するのが**自己平衡木**（AVL木・赤黒木）。
実務ではDBのインデックス（B-Tree）、OS内部の順序付きマップ、Pythonの`sortedcontainers`などで広く使われる。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `key`、`left`、`right` を持つ
- 検索：ルートから比較し左右を辿る
- 挿入：検索して空きノードに追加
- **最悪ケース（偏り木）**：連続昇順挿入でリスト状になりO(n)

```
  1
   \
    2       ← 昇順挿入で退化した例
     \
      3
```

### AVL木

- **バランス条件**：各ノードの左右の高さ差（バランス因子）が `{-1, 0, 1}` であること
- 挿入・削除後に違反したらローテーションで修正（4種類：左・右・左右・右左）
- 高さは常に O(log n) を保証
- 読み取り多用途向き（厳密なバランスにより検索が速い）

### 赤黒木

- 各ノードに**赤or黒**の色属性を持たせる
- 不変条件：
  1. ルートは黒
  2. 赤ノードの子は必ず黒
  3. 任意ノードから葉までの黒ノード数は等しい
- バランスはAVL木より緩い（高さは最悪 `2 * log(n+1)`）
- 挿入・削除時のローテーション回数がAVL木より少ない → **書き込み多用途向き**
- Linux `CFS`スケジューラ、C++ `std::map`、Java `TreeMap` で採用

---

## 計算量・パフォーマンス特性

| 操作       | BST（均衡） | BST（偏り） | AVL木    | 赤黒木   |
|-----------|-----------|-----------|---------|---------|
| 検索       | O(log n)  | O(n)      | O(log n)| O(log n)|
| 挿入       | O(log n)  | O(n)      | O(log n)| O(log n)|
| 削除       | O(log n)  | O(n)      | O(log n)| O(log n)|
| 最悪高さ   | O(n)      | O(n)      | 1.44 log n | 2 log n |

- AVL木は赤黒木より高さが低い → 検索はわずかに速い
- 赤黒木は挿入・削除のローテーションが少ない → 書き込みが速い

---

## コード例（Python: BST の基本操作）

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, key):
        def _insert(node, key):
            if not node:
                return Node(key)
            if key < node.key:
                node.left = _insert(node.left, key)
            elif key > node.key:
                node.right = _insert(node.right, key)
            return node
        self.root = _insert(self.root, key)

    def search(self, key):
        node = self.root
        while node:
            if key == node.key: return True
            node = node.left if key < node.key else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** → ランダム挿入でも最悪ケースは存在する
- **AVL木と赤黒木の使い分け**：検索ヘビーならAVL、挿入/削除ヘビーなら赤黒木
- **削除の実装が複雑**：削除はBSTの中で最も難しく、後継ノード（中順次）への置換が必要
- **Pythonに標準の平衡BSTはない** → `sortedcontainers.SortedList`（外部ライブラリ）か`heapq`で代用する場面が多い
- **B-Treeと混同しない**：DBインデックスに使われるのはBSTではなくB-Tree（多分岐、ディスクI/O最適化）

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連：**

- **Neon（PostgreSQL）のインデックス**：`CREATE INDEX` で生成されるのはB-Tree（赤黒木の派生）。クエリの WHERE/ORDER BY が速くなる仕組みを理解する基盤
- **APIの範囲検索**：`WHERE created_at BETWEEN ? AND ?` が速い理由は、BSTの順序性（中順探索でソート済みを取れる）がB-Treeに引き継がれているから
- **インメモリキャッシュ**：FastAPIでソート済み結果を維持したい場合、`sortedcontainers.SortedList` が O(log n) 挿入でリスト維持
- **Cloud Runのスケーリング判断**：リクエストキューの管理にOSが使う順序付きデータ構造（赤黒木ベース）の計算量が応答時間に影響する
