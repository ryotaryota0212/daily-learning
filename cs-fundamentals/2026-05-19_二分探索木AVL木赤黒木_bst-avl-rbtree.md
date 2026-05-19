# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という順序制約を持つ木構造で、探索・挿入・削除を平均O(log n)で行える。しかし単純なBSTは偏りが生じると最悪O(n)に劣化するため、AVL木・赤黒木などの**自己平衡木**が実務で使われる。Pythonの`sortedcontainers`、PostgreSQLのB-Tree、Java の`TreeMap`はこの原理の上に成り立っている。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左部分木の全値 < 自身 < 右部分木の全値」を満たす
- 探索：根から大小比較を繰り返して目標ノードへ辿る
- 挿入：探索で「落ちた場所」にノードを追加
- **問題点**：昇順データを挿入すると右一直線になり、高さ=n → O(n)

```
昇順挿入の例:
1 → 2 → 3 → 4

1
 \
  2
   \
    3
     \
      4   ← 高さ4, O(n)の劣化
```

### AVL木

- 各ノードで「左右の部分木の高さの差（バランス因子）≤ 1」を保証
- 挿入・削除後に違反が生じたら**回転操作**（左回転・右回転）で修正
- 常に高さ ≈ 1.44 log n → 探索は確実にO(log n)
- **欠点**：回転頻度が高く、書き込み多用時はオーバーヘッド大

### 赤黒木

- ノードに「赤」「黒」の色を付け、4つの制約で高さを制御:
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数は同じ（黒高さ一定）
  4. 葉（NIL）は黒
- 高さ ≤ 2 log(n+1) → O(log n)保証
- AVL木より回転回数が少なく、**挿入・削除が高速**
- Linux kernelのプロセススケジューラ、Java `TreeMap`、C++ `std::map`で採用

---

## 計算量まとめ

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 探索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- **AVL木**：探索が多い場合に有利（より厳密に平衡）
- **赤黒木**：挿入・削除が多い場合に有利（回転コスト低）

---

## コード例（BSTの基本実装）

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

- **「BSTは常にO(log n)」は誤り** — 平衡が保たれないと最悪O(n)になる
- **AVL木が赤黒木より常に速いわけではない** — 読み取り専用ならAVL有利、書き込み多用なら赤黒木有利
- **Pythonの`dict`や`set`はBSTではなくハッシュテーブル** — 順序保証が必要なら`sortedcontainers.SortedList`を使う
- **削除は挿入より複雑** — 子が2つある場合は中順後継ノード（右部分木の最小値）で置き換える

---

## 実務での使いどころ

| シナリオ | 具体例 |
|----------|--------|
| **範囲クエリ** | `BETWEEN`や`ORDER BY`が多いDB列へのインデックス（PostgreSQLのB-Tree） |
| **スコアランキング** | スコア順位管理に`SortedList`（挿入O(log n)、k番目取得O(log n)） |
| **FastAPI + Neon** | Neonのインデックスは内部的にB-Tree。複合インデックスの列順が探索効率を左右する |
| **Cloud Run** | リクエストルーティングや設定値のプレフィックス探索に応用可能 |

**個人開発での実践ポイント：**
- Neonで`created_at`や`score`に範囲検索が多いなら`CREATE INDEX`でB-Treeインデックスを作成
- 頻繁に更新される列よりも、読み取りが多い列のインデックスが効果的（赤黒木 vs AVL木の選択論理と同じトレードオフ）
