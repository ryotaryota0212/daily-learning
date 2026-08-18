# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という不変条件を持つ木構造で、探索・挿入・削除を効率化する。
しかし、素朴なBSTは最悪O(n)に退化するため、実用では**自己平衡木**（AVL木・赤黒木）を使う。
Pythonの`sortedcontainers`、JavaのTreeMap、PostgreSQLのB-Treeインデックスも同じ原理が基盤にある。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `(key, left, right)` を持つ
- 探索: ルートから「大小比較で左右に進む」→ 木の高さ分だけ比較
- 挿入・削除: 探索で位置を特定し、ポインタを付け替え
- **問題点**: ソート済みデータを挿入するとリスト状になり高さがO(n)

```
挿入順: 1, 2, 3, 4, 5
  1
   \
    2
     \
      3   ← 高さ=N、探索O(N)になる
```

### AVL木（高さ平衡木）

- 各ノードに「左右の高さの差（バランス因子）」を記録
- **バランス因子が ±2 になった瞬間に回転（Rotation）で修正**
- 常に高さ ≤ 1.44 × log₂N を保証
- 回転の種類: 左回転・右回転・左右回転・右左回転

```
右回転の例（右に偏った場合の逆）:
    3           2
   /           / \
  2    →      1   3
 /
1
```

### 赤黒木

- 各ノードに「赤 or 黒」の色を付ける
- **色に関する4つの制約**を維持することで「どの葉へのパスも同じ黒ノード数」を保証:
  1. ノードは赤か黒
  2. ルートは黒
  3. 赤ノードの子は必ず黒（赤が連続しない）
  4. 全リーフ（NIL）への黒ノード数は等しい
- 高さ ≤ 2 × log₂(N+1) を保証
- AVL木より**回転回数が少なく、挿入・削除が高速**
- 探索はAVL木がわずかに速い（AVLの方が厳密に平衡）

---

## 計算量・パフォーマンス特性

| 操作     | 素朴BST（平均） | 素朴BST（最悪） | AVL木   | 赤黒木  |
|----------|----------------|----------------|---------|---------|
| 探索     | O(log N)       | O(N)           | O(log N)| O(log N)|
| 挿入     | O(log N)       | O(N)           | O(log N)| O(log N)|
| 削除     | O(log N)       | O(N)           | O(log N)| O(log N)|
| 空間     | O(N)           | O(N)           | O(N)    | O(N)    |

- AVL木: 探索が多い読み取り重視ワークロードに向く
- 赤黒木: 挿入・削除が多い書き込み重視ワークロードに向く（Linux カーネル、Java TreeMap で採用）

---

## コード例（Python: 簡略BST）

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
            if node is None:
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
            if key == node.key:
                return True
            node = node.left if key < node.key else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log N)」は誤り** → 平衡が保たれていない場合はO(N)
- **AVL木と赤黒木を実装から使い分ける必要はほぼない** → 言語標準ライブラリが適切な方を選択済み
- **Pythonの`dict`や`set`はBSTでなくハッシュテーブル** → ソート順が必要なら`sortedcontainers.SortedList`を使う
- **削除はBSTで最も複雑** → 「後継ノード（右部分木の最小値）と交換してから削除」というパターンが定石
- **B-TreeはBSTとは別物** → データベースインデックスで使われるB-Treeは複数キーを1ノードに持つ多分木

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **Neonのインデックス**: `CREATE INDEX` で作られるB-Treeインデックスは赤黒木と同じ原理。`WHERE id > 100 ORDER BY id` のような範囲検索が速い理由
- **ソート済みデータの管理**: ユーザーランキングや時系列データを`sortedcontainers.SortedList`で管理するとO(log N)挿入・O(1)最小/最大取得が可能
- **FastAPIのルーティング**: フレームワーク内部でルートをトライ木（BSTの変形）で管理し高速マッチング
- **実装よりも「どこで使われているか」の理解が重要** → DBのEXPLAIN出力でIndex Scanが出たとき、背後でBSTが動いていると知ることでチューニングの判断ができる
