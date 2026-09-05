# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

木構造の中でも「探索に特化した自己平衡木」は、データベースインデックスからOSスケジューラまで幅広く使われる基盤技術。  
- 単純なBSTは最悪O(n)に退化するが、AVL木・赤黒木は常にO(log n)を保証
- PostgreSQLのB-Tree、C++の`std::map`、JavaのTreeMapなど主要実装の理解に直結
- 「なぜSELECTが遅いか」をインデックスレベルで考えるための前提知識

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは「左子 < 自ノード < 右子」の順序性を保持
- 探索・挿入・削除はルートから比較しながら木を降下するだけ
- **致命的な問題**: ソート済みデータを順に挿入すると一本道のリストに退化 → O(n)

```
挿入: 1, 2, 3, 4, 5

1
 \
  2
   \
    3  ← 探索がO(n)になる
     \
      4
       \
        5
```

### AVL木
- 各ノードで `|左部分木の高さ - 右部分木の高さ| ≤ 1` を常に維持
- 挿入・削除後に違反した場合「回転」操作でリバランス
  - **LL回転・RR回転**（単純回転）
  - **LR回転・RL回転**（二重回転）
- 厳密にバランスするため探索は高速、ただし挿入・削除で回転が多発

### 赤黒木
- 各ノードに「赤」か「黒」のカラーを付与し、5つの制約で準平衡を保証

| 制約 | 内容 |
|------|------|
| 1 | 全ノードは赤か黒 |
| 2 | ルートは黒 |
| 3 | 赤ノードの子は必ず黒（赤の連続禁止） |
| 4 | 任意のノードからNIL葉までの黒ノード数は同じ |
| 5 | NIL葉は黒 |

- AVL木より「緩い」バランスだが1回の挿入/削除での回転は**最大2回**
- **実際の採用例**: Linux CFSスケジューラ、C++ `std::map`/`set`、Java `TreeMap`

---

## 計算量

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転回数 | — | — | 多い（O(log n)回まで） | 最大2回 |

**使い分けの指針**
- 探索が多い → AVL木（厳密平衡で木が低い）
- 挿入・削除が多い → 赤黒木（回転コストが低い）

---

## コード例（Python: BST の挿入と探索）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def insert(self, root, val):
        if not root:
            return Node(val)
        if val < root.val:
            root.left = self.insert(root.left, val)
        else:
            root.right = self.insert(root.right, val)
        return root

    def search(self, root, val):
        if not root or root.val == val:
            return root
        if val < root.val:
            return self.search(root.left, val)
        return self.search(root.right, val)
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ランダムデータなら平均O(log n)だが、ソート済み入力で必ずO(n)に退化
- **「AVLは常に最速」は誤り**: 書き込みが多い場合、回転コストで赤黒木より遅くなることがある
- **自前実装は危険**: 赤黒木のカラーリング・回転の実装は複雑でバグが混入しやすい。実務ではライブラリ依存が正解
- **BST ≠ B-Tree**: データベースのB-Treeは「多分木」でBSTとは別物（ただし考え方は共通）

---

## 実務での使いどころ

**FastAPI + Neon（PostgreSQL）との関連**
- PostgreSQLのインデックスはB-Tree（内部は赤黒木的な平衡木）
- `EXPLAIN ANALYZE` でインデックスが使われているか確認する習慣が重要
- 複合インデックスはカラムの順序でO(log n)が効くか変わる（先頭カラムから順に絞り込む）

```sql
-- インデックスが効くケース
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1 AND created_at > '2026-01-01';
-- user_id, created_at の複合インデックスがあればBitmap Index Scanになる
```

**Cloud Run との関連**
- インメモリで順序付きデータを扱う場合、Pythonの`sortedcontainers.SortedList`（内部は準平衡木）が有用
- 範囲クエリ（「100〜200の間の値を全部取得」）は平衡木が配列より圧倒的に効率的
