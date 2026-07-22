# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）はデータを順序付きで保持し、O(log n)での探索・挿入・削除を目指す基本データ構造。
ただし普通のBSTは入力順によって偏り、最悪O(n)に劣化する。
AVL木・赤黒木はそれぞれ「高さのバランス」を自動的に保つ自己平衡BST。
PostgreSQLやLinuxカーネルのスケジューラも赤黒木を採用しており、実務上の重要性は高い。

---

## 仕組みの要点

### 二分探索木（BST）の基本
- 各ノード：左の子 < 自分 < 右の子
- 探索：根から比較しながら左右に降りる
- 挿入：探索で見つけた空き位置に追加
- **問題点**：昇順で挿入すると完全に右に偏った「線形リスト」になりO(n)

```
昇順挿入の例（1,2,3,4,5）:
1
 \
  2
   \
    3
     \
      4
       \
        5
```

### AVL木
- **制約**：各ノードの左右の高さの差が最大1
- 挿入・削除後に**回転操作**（左回転・右回転・左右回転・右左回転）で修正
- 高さ：常にO(log n)を保証
- 探索は高速だが、回転が頻繁なため**書き込みコストが高め**

### 赤黒木
- 各ノードを赤か黒に色付け、以下の制約を維持：
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 根から全ての葉（NILノード）までの黒ノード数が等しい
- 高さ：最大 2log(n+1)（AVLより緩い制約）
- **回転・色変えの回数が少なく**、挿入・削除が高速
- 実装：Java `TreeMap`、Linux `CFS`スケジューラ、PostgreSQLのMVCC管理

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転数（挿入） | - | - | 最大2回 | 最大2回 |
| 回転数（削除） | - | - | O(log n)回 | 最大3回 |

**使い分けの指針**
- 読み取り多め → AVL木（高さ厳格、探索がわずかに速い）
- 書き込み多め → 赤黒木（回転が少ない）

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
        def _insert(node, val):
            if not node:
                return BSTNode(val)
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

- **「BSTは常にO(log n)」は誤り**：バランスが保証されていないと最悪O(n)
- **「AVLの方が常に速い」は誤り**：削除が多い場合は赤黒木の方が有利
- **インオーダートラバーサルは昇順**：これを利用してソート済み配列的に使える
- **削除は3ケースある**：葉ノード・子が1つ・子が2つ（後継者ノードで置換）
- 赤黒木の自前実装は複雑なため、実務ではPythonの`sortedcontainers`等を使う

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**

- **PostgreSQL（Neon）のインデックス**：BツリーがデフォルトだがBSTの原理を理解すると読み替えやすい
- **範囲クエリの効率**：`WHERE id BETWEEN 100 AND 200`はBST的な構造が有効
- **ORMのソート**：`ORDER BY`の内部でインデックスが木構造を活用している
- **将来の最適化**：クエリが遅い場合、実行計画（`EXPLAIN ANALYZE`）で`Index Scan`vs`Seq Scan`を判断するためにBST原理が役立つ
- **キャッシュの実装**：有効期限付きキャッシュをTTL順に管理するとき自己平衡BSTが有用（Python `sortedcontainers.SortedList`）
