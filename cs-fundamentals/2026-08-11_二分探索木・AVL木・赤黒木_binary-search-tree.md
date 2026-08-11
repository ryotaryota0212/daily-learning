# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

木構造の検索・挿入・削除を効率化するデータ構造。ハッシュテーブルと異なり「順序」を保持できるため、範囲検索や順序付き列挙が必要な場面で活躍する。PostgreSQLのB-Treeインデックス、Pythonの`sortedcontainers`、Java・C++の`TreeMap`/`std::map`の根幹をなす概念。

---

## 仕組みの要点

### 二分探索木（BST: Binary Search Tree）

- **不変条件**: 左の子 < 親 < 右の子（全ノードで成立）
- 検索: ルートから「大小比較→左右分岐」を繰り返す
- **問題点**: 挿入順が偏ると木が「棒状」になり、最悪O(n)に退化する

```
  挿入順: 5, 3, 7, 2, 4
        5
       / \
      3   7
     / \
    2   4
  ← バランスが取れている（O(log n)）

  挿入順: 1, 2, 3, 4, 5（昇順）
  1 → 2 → 3 → 4 → 5  ← 退化した木（O(n)）
```

### AVL木（高さバランス木）

- **条件**: 各ノードで「左右の高さの差（バランス因子）≤ 1」を常に維持
- 挿入・削除後に「回転（Rotation）」操作でバランスを修復
  - 単回転（左回転・右回転）
  - 二重回転（左右・右左）
- **特徴**: 厳密なバランス保証 → 検索が高速だが、挿入・削除のコスト高め

### 赤黒木（Red-Black Tree）

- **5つの不変条件**（要点のみ）:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 赤ノードの子は黒（赤が連続しない）
  4. どのノードからも葉（NIL）までの黒ノード数が等しい
- AVL木より「緩い」バランス → 回転回数が少なく挿入・削除が速い
- 実用例: Linux kernel のタスクスケジューラ、C++ `std::map`、Java `TreeMap`

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 回転数（挿入時） | - | - | 最大2回 | 最大2回 |
| 回転数（削除時） | - | - | O(log n)回 | 最大3回 |

- **AVL vs 赤黒木の使い分け**:
  - 検索が圧倒的に多い → AVL木（より厳密なバランス）
  - 挿入・削除が多い → 赤黒木（少ない回転で済む）

---

## コード例（Python: 基本的なBST）

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

- **「BSTは常にO(log n)」は誤り**: バランスが保証されるのはAVL木・赤黒木のみ
- **ハッシュテーブルの代替にはならない**: 範囲検索が不要ならハッシュの方が高速（O(1)）
- **削除は挿入より複雑**: 子が2つある場合、「中順後継者（右部分木の最小値）」で置換する手順が必要
- **Pythonの`sortedcontainers.SortedList`**: 内部はB-Treeに近い実装（純粋なBSTではない）

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX`で作られるB-Treeは赤黒木の概念を拡張したもの。範囲クエリ（`WHERE created_at BETWEEN ...`）が高速な理由はここにある
- **APIの並び替え・範囲フィルタ**: DBインデックスが使われているか`EXPLAIN ANALYZE`で確認する際、B-Tree使用を前提に考える
- **Cloud Runのサービス検出**: 内部的にはConsistentHashingやTree構造でノード管理が行われる
- **Pythonでの利用**: `from sortedcontainers import SortedList` → 挿入しながら順序を保ちたい場合（リアルタイムランキングなど）に使う

```python
# FastAPIでのユースケース例: ソート済みリストで上位K件を効率的に管理
from sortedcontainers import SortedList

scores = SortedList()  # 挿入のたびに自動ソート O(log n)
scores.add(85)
scores.add(92)
scores.add(78)
top3 = list(scores[-3:])  # 上位3件 O(k)
```
