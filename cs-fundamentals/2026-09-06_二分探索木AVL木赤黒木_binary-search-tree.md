# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、探索・挿入・削除をO(log n)で実現する。  
ただし、偏った挿入順ではO(n)に劣化する。AVL木・赤黒木はこの劣化を防ぐ「自己平衡木」。  
PythonのSortedContainers、JavaのTreeMap、PostgreSQLのB-Treeはこれらの応用。  
「なぜO(log n)なのか」を理解することで、インデックス設計やアルゴリズム選択の判断軸になる。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは `left < node < right` の制約を満たす
- 探索: 根から比較しながら左右どちらかへ辿る
- 挿入: 探索で見つけた空きにノードを追加
- 削除: 後継ノード（右部分木の最小）で置き換える
- **問題点**: `1→2→3→4→5` の順に挿入すると連結リストと同じ形（O(n)）

### AVL木
- 左右の高さの差（バランス因子）が常に `-1, 0, +1` になるよう保つ
- 挿入・削除後に **回転（rotation）** で再平衡化
  - 右回転・左回転・左右回転・右左回転の4パターン
- 厳密に平衡 → 探索が速い、但し挿入・削除のオーバーヘッドが大きい

### 赤黒木
- 各ノードに「赤」「黒」の色を付け、以下の制約で疎な平衡を保つ:
  1. 根は黒
  2. 赤ノードの子は黒（赤が連続しない）
  3. 任意の根→葉パスの黒ノード数が同じ（黒高さが一定）
- AVL木より平衡が緩い → 回転回数が少なく挿入・削除が速い
- `std::map`（C++）、JavaのTreeMap、Linuxカーネルのタスクスケジューラが採用

---

## 計算量・パフォーマンス特性

| 操作 | BST平均 | BST最悪 | AVL木 | 赤黒木 |
|------|--------|--------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転数（挿入時） | - | - | O(log n) | O(1) |

**使い分けの目安**
- 探索が多い → AVL木（より厳密に平衡）
- 挿入・削除が多い → 赤黒木（回転が少ない）
- 順序付き操作が不要 → ハッシュテーブル（平均O(1)）

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
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ソート済みデータを挿入すると最悪O(n)
- **AVL木・赤黒木の実装は難しい**: 実務ではライブラリを使う（自前実装不要）
- **ハッシュテーブルとの混同**: BSTは順序付き操作（範囲検索・最小/最大）が得意。ハッシュは順序不要の単純探索が得意
- **削除の複雑さ**: 後継ノードによる置き換えが必要で、単純ではない

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連**

- **PostgreSQL（Neon）のインデックス**: B-Treeインデックスは赤黒木の多分木版。`WHERE id > 100 AND id < 200` の範囲検索が速い理由はここにある
- **APIの順序付きレスポンス**: `ORDER BY` が速く効くのはB-Treeインデックスで値が整列されているから
- **SortedContainers（Python）**: 優先度キューの代替として範囲操作が必要な場合に使用
- **キャッシュのキー管理**: 有効期限順で管理する際、min-heapより範囲削除が容易
