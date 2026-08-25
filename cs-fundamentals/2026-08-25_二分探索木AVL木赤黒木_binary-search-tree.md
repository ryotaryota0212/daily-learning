# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 自分 < 右の子」という順序性を持つ木構造。検索・挿入・削除が O(log n) になる（バランスが保たれた場合）。データが偏ると O(n) に劣化するため、自己平衡木（AVL木・赤黒木）が実用される。実務では DB のインデックス、言語の標準ライブラリ（Python の sortedcontainers、Java の TreeMap）の内部実装として重要。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは「左部分木 < ノード値 < 右部分木」を満たす
- 検索：根から比較しながら左右を選んで降りる
- 挿入：検索と同じ経路で末端に追加
- 削除：子の数によって処理が異なる
  - 子なし → 削除
  - 子が1つ → 子で置き換え
  - 子が2つ → 右部分木の最小値（後継ノード）と入れ替えてから削除
- **問題点**：ソート済みデータを順に挿入すると木が直線状になり、O(n) に劣化

### AVL木

- 各ノードで「左右の高さの差（バランス因子）≤ 1」を維持
- 挿入・削除後に違反が生じたら **回転** で修正
  - 右回転・左回転・左右回転・右左回転（4パターン）
- 回転コストは O(1) / 挿入・削除あたり最大 O(log n) 回の回転
- 高さが厳密に管理されるため **検索が赤黒木より速い**

### 赤黒木

- 各ノードに「赤/黒」の色属性を追加し、以下の制約で緩やかなバランスを保つ
  1. 根は黒
  2. 赤ノードの子は黒（赤が連続しない）
  3. どの葉（NIL）への経路も同じ黒ノード数（黒高さ一致）
- 挿入・削除後の修正操作が AVL より少ない → **書き込み頻度が高い場合に有利**
- Linux カーネルのスケジューラ・C++ の `std::map` などに使用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転数（挿入） | - | - | O(log n) | O(1) 償却 |

- AVL：高さ ≤ 1.44 log₂(n) → 検索最速
- 赤黒木：高さ ≤ 2 log₂(n+1) → 挿入・削除が少ない回転で済む

---

## コード例（Python：BST の検索と挿入）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _insert(node, v):
            if not node:
                return Node(v)
            if v < node.val:
                node.left = _insert(node.left, v)
            elif v > node.val:
                node.right = _insert(node.right, v)
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

- **「BST は常に O(log n)」は誤り**：バランスが崩れると O(n) になる。ランダムデータでは平均 O(log n) だが、挿入順が偏ると最悪ケースになる
- **AVL木 vs 赤黒木の選択**：読み込み多 → AVL、書き込み多 → 赤黒木。多くの標準ライブラリは赤黒木を採用
- **削除は難しい**：後継ノード（中順次ノード）の概念を誤解しやすい。左部分木の最大値ではなく **右部分木の最小値** が後継
- **Python の `sortedcontainers.SortedList`**：内部は B-tree 系で、BST より実務的に速いことが多い

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のインデックス**：B-Tree インデックスは赤黒木と同じ原理でバランスを保つ。`CREATE INDEX` を理解するうえで BST の概念が基礎になる
- **API の範囲クエリ**：`WHERE price BETWEEN 1000 AND 5000` のような範囲クエリは木の中順走査で効率的に実行される。インデックスがない場合は全スキャンになる
- **ランキング・リアルタイムソート**：スコアボードや価格帯フィルタなど、動的に順序を保ちながら検索が必要な場面では BST 系データ構造が有効
- **FastAPI でのソート済みデータ返却**：DB に任せるのが基本だが、インメモリキャッシュで順序付きデータを扱う場合に `SortedList` が役立つ
