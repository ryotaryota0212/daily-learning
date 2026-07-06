# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

順序付きデータを高速に検索・挿入・削除するための木構造。ハッシュテーブルは順序を持たないが、木は「範囲検索」や「ソート済み順アクセス」が可能。PostgreSQLのB-Treeインデックスはこの系譜の発展形であり、原理を理解するとクエリチューニングに直結する。

---

## 仕組みの要点

### 二分探索木（BST: Binary Search Tree）

```
      8
     / \
    3   10
   / \    \
  1   6    14
     / \
    4   7
```

- **不変条件**: 左の子 < 親 < 右の子
- **検索**: ルートから大小比較で左右を選択
- **問題点**: 挿入順によっては片寄りが生じる（最悪 O(n) の線形リスト）

```python
class Node:
    def __init__(self, val):
        self.val, self.left, self.right = val, None, None

def insert(root, val):
    if not root:
        return Node(val)
    if val < root.val:
        root.left = insert(root.left, val)
    else:
        root.right = insert(root.right, val)
    return root

def search(root, val):
    if not root or root.val == val:
        return root
    return search(root.left, val) if val < root.val else search(root.right, val)
```

---

### AVL木（自己平衡BST）

- **高さバランス条件**: 各ノードの左右部分木の高さ差（バランス因子）が -1, 0, 1 のいずれか
- 挿入・削除後に違反が生じたら**回転（rotation）**で修正
  - 右回転、左回転、左右回転、右左回転の4パターン
- 常に高さが O(log n) に保たれるため、検索・挿入・削除すべて O(log n) 保証

**回転のイメージ（右回転）**:
```
    z              y
   /      →       / \
  y              x   z
 /
x
```

---

### 赤黒木（Red-Black Tree）

- 各ノードに赤/黒の色を付与し、以下のルールで高さを管理:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードからNILまでの黒ノード数は全経路で同じ
- AVL木より制約が緩く、**回転回数が少ない**（挿入: 最大2回、削除: 最大3回）
- JavaのTreeMap、C++のstd::map、Linuxのスケジューラで採用

---

## 計算量・パフォーマンス特性

| 操作       | BST（平均）| BST（最悪）| AVL木     | 赤黒木    |
|-----------|-----------|-----------|----------|----------|
| 検索       | O(log n)  | O(n)      | O(log n) | O(log n) |
| 挿入       | O(log n)  | O(n)      | O(log n) | O(log n) |
| 削除       | O(log n)  | O(n)      | O(log n) | O(log n) |
| 空間       | O(n)      | O(n)      | O(n)     | O(n)     |

- **AVL木 vs 赤黒木**: AVL木の方が高さが低い（検索が若干速い）。赤黒木の方が挿入・削除が速い（回転コストが低い）
- 読み込みが多い場合はAVL木、書き込みが多い場合は赤黒木が有利

---

## よくある誤解・落とし穴

- **「BSTはO(log n)」は保証ではない**: ソート済みデータを順に挿入すると線形リストになりO(n)。必ず平衡木か別の対策が必要
- **削除が最も複雑**: 削除するノードに子が2つある場合、中順後継者（in-order successor）で置き換える処理が必要
- **赤黒木の実装は難しい**: 自前実装は避け、言語標準ライブラリの`SortedDict`や`sortedcontainers`を使う
- **Pythonの`heapq`はBSTではない**: ヒープは優先度キュー、BSTは順序付きマップ。用途が異なる

---

## 実務での使いどころ

**FastAPI + Neon(PostgreSQL) + Cloud Run スタックとの関連**

- **PostgreSQL の B-Tree インデックス**: 赤黒木の発展形（ページ単位のディスクI/Oに最適化）。`EXPLAIN ANALYZE` でインデックスが効いているか確認できる
- **範囲検索に強い**: `WHERE created_at BETWEEN '2026-01-01' AND '2026-06-30'` はB-Treeインデックスで効率的に処理される（ハッシュインデックスでは不可）
- **Pythonでの実用**: `sortedcontainers.SortedList` がPython製の平衡BST相当。順序付き検索が必要な場面で`list`の代替として使える
- **実装不要、原理は必須**: 自前実装は不要だが「なぜインデックスが範囲検索に強いか」を説明できることがデバッグ・設計判断に直結する
