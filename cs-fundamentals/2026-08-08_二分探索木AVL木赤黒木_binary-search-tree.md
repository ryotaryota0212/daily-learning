# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「検索・挿入・削除をO(log n)で行う」ための基礎データ構造。  
ただし単純なBSTは偏りによりO(n)に劣化するため、自己平衡木（AVL木・赤黒木）が実務で使われる。  
PostgreSQLのB-Treeインデックス、言語標準ライブラリのSortedSet、Redisのzset等の根幹にある考え方。

---

## 仕組みの要点

### 二分探索木（BST）の基本ルール
- 各ノードは左 < 自身 < 右を満たす
- 中順探索（左→自身→右）でソート済みリストを得られる
- 偏った挿入順（昇順など）で木が線形になりO(n)に劣化

```
     5
    / \
   3   7
  / \   \
 1   4   9
```

### AVL木
- 各ノードの左右の高さの差（バランス因子）を常に -1〜1 に保つ
- 挿入・削除後に違反があれば**回転（Rotation）**で修正
  - 単回転（LL回転 / RR回転）
  - 二重回転（LR回転 / RL回転）
- 高さを厳密に保つため検索が高速だが、挿入・削除のコストがやや高い

### 赤黒木（Red-Black Tree）
- ノードに赤・黒のラベルを持たせる。以下の制約を守る:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. どのノードからNULL葉までの黒ノード数が等しい（黒高さ一定）
- AVL木より制約が緩い → 回転回数が少なく**挿入・削除が高速**
- C++の`std::map`、JavaのTreeMap、Linuxカーネルのスケジューラで採用

---

## 計算量・パフォーマンス特性

| 操作     | 単純BST（平均/最悪） | AVL木      | 赤黒木     |
|--------|--------------|-----------|-----------|
| 検索     | O(log n)/O(n) | O(log n)  | O(log n)  |
| 挿入     | O(log n)/O(n) | O(log n)  | O(log n)  |
| 削除     | O(log n)/O(n) | O(log n)  | O(log n)  |
| 空間     | O(n)          | O(n)      | O(n)      |

- AVL木：高さが最大 1.44 log(n+2)
- 赤黒木：高さが最大 2 log(n+1)（AVL木より背が高い場合あり）
- 検索頻度が高いならAVL木、書き込み頻度が高いなら赤黒木が有利

---

## コード例（Python：単純BSTの検索・挿入）

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
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り** — ソート済みデータを挿入すると高さがO(n)になる
- **AVL木と赤黒木の選択**：実装難易度は赤黒木の方が高い。実務では既存ライブラリを使うので内部方式を意識する必要は少ない
- **削除の複雑さ**：後継ノード（右部分木の最小値）での置換が必要。平衡木ではさらに回転が絡む
- **ハッシュテーブルとの比較**：BSTは順序付きイテレーション・範囲検索が可能。ハッシュはO(1)だが順序を持たない

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **Neon（PostgreSQL）のインデックス**：通常のBTREEインデックスはB-Tree（バランス木の多分木版）。`WHERE price BETWEEN 100 AND 500` のような範囲検索はインデックスを活かせる
- **ソート済み結果のクエリ設計**：`ORDER BY`の列にインデックスがあるとIndex Scanで効率化。BSTの中順探索と同じ原理
- **APIのソート・フィルタ**：FastAPIでクエリパラメータによるソートを実装する際、DB側のインデックスがなければPython側でソート（O(n log n)）が必要になる判断が自然にできる
- **優先度付きタスクキュー**：非同期タスクのスケジューリングにはヒープ（次回）かBSTベースのSortedContainersが有効
