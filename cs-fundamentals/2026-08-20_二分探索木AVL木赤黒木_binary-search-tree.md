# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序不変条件を持つ木構造で、検索・挿入・削除を平均 O(log n) で実現する。しかし素朴な BST は入力順次第で O(n) に退化するため、AVL 木や赤黒木が「自己平衡」機能を加えて最悪 O(log n) を保証する。実務では sorted set / ordered map の実装基盤（Python の `sortedcontainers`、Java の `TreeMap`、PostgreSQL の B-tree インデックス）として現れる。

---

## 仕組みの要点

### 二分探索木（BST）
- **不変条件**: 各ノードの左サブツリーは全て小さく、右は全て大きい
- 検索: ルートから「大小比較 → 左右に降りる」を繰り返す
- 挿入: 検索と同じ経路の末端に追加
- 削除: 後継ノード（右サブツリーの最小値）と置き換えてから削除
- **問題点**: ソート済みデータを挿入すると一本鎖（高さ n）に退化 → O(n)

### 木の高さと計算量の関係
```
バランスが取れている場合:
高さ h ≈ log₂ n  → 操作コスト O(log n)

退化した場合（一本鎖）:
高さ h = n       → 操作コスト O(n)
```

### AVL木
- 各ノードで「左右のサブツリーの高さの差（バランス因子）≤ 1」を維持
- 挿入・削除後にバランス因子が ±2 になったら**回転**で修正
  - 右回転・左回転（単回転）
  - 左右回転・右左回転（二重回転）
- **特性**: 高さが厳密に ≤ 1.44 log₂ n に収まるため検索が最速
- **代償**: 挿入・削除のたびに多くの回転が発生することがある

### 赤黒木
- 各ノードに「赤」か「黒」の色を付け、以下の不変条件を維持:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの経路上の黒ノード数が等しい（黒高さ一定）
- 高さの上限: 2 log₂(n+1) → AVL より緩いがまだ O(log n)
- **特性**: AVL より回転回数が少ない → 挿入・削除が速い
- 実装例: Linux カーネルの CFS スケジューラ、Java `TreeMap`、C++ `std::map`

### 回転の概念図
```
左回転 (x を根とする部分木):
      x              y
     / \    →       / \
    A   y          x   C
       / \        / \
      B   C      A   B
```

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木       | 赤黒木      |
|----------|-------------|-------------|-------------|-------------|
| 検索     | O(log n)    | O(n)        | O(log n)    | O(log n)    |
| 挿入     | O(log n)    | O(n)        | O(log n)    | O(log n)    |
| 削除     | O(log n)    | O(n)        | O(log n)    | O(log n)    |
| 空間     | O(n)        | O(n)        | O(n)        | O(n)        |

- **AVL 木の回転回数**: 挿入 O(1)、削除 O(log n) 回まで
- **赤黒木の回転回数**: 挿入 ≤ 2 回、削除 ≤ 3 回（定数）

---

## コード例（Python: 素朴な BST）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, v):
            if not node:
                return Node(v)
            if v < node.val:
                node.left = _ins(node.left, v)
            elif v > node.val:
                node.right = _ins(node.right, v)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val:
                return True
            node = node.left if val < node.val else node.right
        return False
```

実務では `from sortedcontainers import SortedList` を使う方が実装コストゼロで効率的。

---

## よくある誤解・落とし穴

- **「BST は常に O(log n)」は誤り**: ソート済みデータを挿入すると O(n) に退化する
- **「AVL 木が常に最速」は誤り**: 読み取り中心なら AVL が有利だが、書き込みが多い場合は赤黒木の方が有利（回転回数が少ないため）
- **二重回転の見落とし**: 「左右不均衡」と「右左不均衡」は異なるパターンで、それぞれ対応する二重回転が必要
- **削除の複雑さ**: 子が 2 つあるノードの削除は「後継ノードを探す → 値を置き換え → 後継を削除」の 3 ステップで実装ミスが多い
- **ハッシュテーブルと混同**: BST は順序を持つ（範囲クエリが可能）が、ハッシュは順序なし（定数時間）という用途の違いを意識する

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

**PostgreSQL の B-tree インデックス**
- Neon（PostgreSQL）のデフォルトインデックスは B-tree（赤黒木の原理を多分岐に拡張）
- `WHERE age BETWEEN 20 AND 30` のような範囲クエリが O(log n) で動くのは BST の順序性のおかげ
- ハッシュインデックス（= 相当のみ）との使い分けが重要

```sql
-- 範囲クエリに強い B-tree（デフォルト）
CREATE INDEX idx_age ON users(age);

-- 等価クエリ専用でさらに高速なハッシュ
CREATE INDEX idx_email ON users USING hash(email);
```

**FastAPI でのソート済みデータ処理**
- ランキング・リーダーボード・タイムライン系の機能で「挿入しながら順序を保つ」必要がある場合に `SortedList` を使うと BST の恩恵を受けられる
- ただしインメモリ保持が前提なので Cloud Run のステートレス性に注意（スケールアウト時にメモリが共有されない）
