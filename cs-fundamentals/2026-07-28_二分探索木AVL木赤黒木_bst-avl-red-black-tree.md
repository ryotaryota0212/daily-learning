# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、検索・挿入・削除をO(log n)で行える。
ただし偏りが生じると最悪O(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用では使われる。
PythonのSortedContainersや多くのDB実装の内部でこれらの概念が使われており、「なぜO(log n)か」を理解することが計算量を直感的に把握する力になる。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは左部分木 < ノード値 < 右部分木
- 検索：ルートから比較して左右に絞り込む → 木の高さ分だけたどる
- **弱点**：昇順挿入すると高さ = n（線形リスト化）でO(n)に劣化

```
挿入順: 1, 2, 3, 4, 5 → 右に伸びるだけの棒状ツリー
```

### AVL木
- 任意ノードの左右部分木の高さ差（バランス因子）を **±1以内** に維持
- 挿入・削除時に回転（Rotation）でバランスを修正
  - **右回転・左回転・左右回転・右左回転** の4パターン
- 高さが常にO(log n)に保たれる → 検索が安定して速い
- 欠点：回転が頻繁に起きる → 書き込み多用途ではオーバーヘッド大

### 赤黒木（Red-Black Tree）
- 各ノードに赤/黒の色を持ち、以下の制約でゆるやかに平衡を保つ：
  1. ルートは黒
  2. 赤ノードの子は必ず黒（連続赤はNG）
  3. どのノードからNull葉までの黒ノード数は同じ（黒高さの一致）
- AVLより高さ上限がわずかに緩い（最大 2log n）が回転回数が少ない
- **実用例**：Linux の CFSスケジューラ、C++ `std::map`、Java `TreeMap`

---

## 計算量・パフォーマンス特性

| 操作       | BST（最悪）| AVL木    | 赤黒木   |
|-----------|-----------|---------|---------|
| 検索       | O(n)      | O(log n)| O(log n)|
| 挿入       | O(n)      | O(log n)| O(log n)|
| 削除       | O(n)      | O(log n)| O(log n)|
| 回転コスト  | なし       | 最大O(log n)回 | 最大3回 |

- AVL木：検索重視（厳格に平衡）
- 赤黒木：挿入・削除重視（回転少なく実装コスト低い）

---

## コード例（Python: BST の検索・挿入）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, val):
            if not node:
                return Node(val)
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

- **「BSTはO(log n)」は平均の話** → 最悪はO(n)。自己平衡木を使わないと本番で詰まる
- 赤黒木の実装は複雑だが、**自分で実装する必要はほぼない**（標準ライブラリを使う）
- AVL木は高さが厳格なため、検索が多くて書き込みが少ないケース（読み取り専用インデックス等）に向く
- Pythonに組み込みの平衡木はない → `sortedcontainers.SortedList` が実用的

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連：**

- **Neon（PostgreSQL）のB-Treeインデックス**：内部で赤黒木に近いB-Treeを使用。インデックス設計の計算量を理解する土台になる
- **範囲検索が必要なクエリ**（`BETWEEN`, `ORDER BY`）：BSTの順序特性があるからこそ効率的に動く
- **FastAPIでのソート済みデータ管理**：メモリ上で順序付きコレクションが必要なら `SortedList` を検討
- **Cloud Run上でのキャッシュ管理**：有効期限付きキャッシュの順序管理に優先度付きキューや平衡木が使われる

> 「なぜDBのインデックスで範囲検索が速いのか」がBSTの原理から説明できる。
