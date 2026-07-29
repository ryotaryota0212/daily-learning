# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は、検索・挿入・削除を効率的に行うための木構造データ構造。  
ハッシュテーブルと異なり、**順序付きデータの取得**（範囲検索・最小/最大・ソート済み列挙）が得意。  
PostgreSQLのB-Treeインデックスや標準ライブラリのsorted setの内部実装に使われており、DB設計やパフォーマンスチューニングの理解に直結する。

---

## 仕組みの要点

### 二分探索木（BST）の基本ルール

- 各ノードは左・右の子を持つ
- **左の子 < 親ノード < 右の子** が常に成立
- 中順巡回（左→自分→右）でソート済み順に列挙できる

```
      10
     /  \
    5    15
   / \     \
  3   7    20
```

### 問題点：木の偏り

- 挿入順が悪いと木が一直線になる（最悪ケース）
- `1 → 2 → 3 → 4` の順で挿入すると右に偏った連結リストと同じになる
- 検索が O(n) に劣化する

### AVL木：厳格な高さバランス

- 各ノードで **左右の高さの差（バランス因子）が -1, 0, 1** を維持
- 違反したらローテーション（左回転・右回転）で修正
- 挿入・削除ごとに厳格に再バランス → **検索が高速**、書き込みコストはやや高い

### 赤黒木：緩やかなバランス

- 各ノードを赤か黒に色付け
- 以下の性質を維持することで高さが `2 * log(n)` 以内に収まる:
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数が等しい
- AVL木より再バランスの頻度が低い → **挿入・削除が高速**
- Linux カーネルのプロセス管理、Java の TreeMap、C++ の std::map に採用

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- **AVL木** は高さが厳密に `1.44 * log(n)` 以内 → 検索回数が少ない
- **赤黒木** は高さが最大 `2 * log(n)` → 検索は AVL より遅いが、挿入・削除のローテーション回数が定数（O(1)）
- 書き込み多 → 赤黒木、読み込み多 → AVL木が適する

---

## コード例（Python：基本 BST の検索・挿入）

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

Pythonの標準ライブラリには BST がないが、`sortedcontainers.SortedList` が赤黒木相当の実装を提供。

---

## よくある誤解・落とし穴

- **「BST は常に O(log n)」→ 誤り**。バランスが崩れると O(n)。本番コードで生の BST を使う場面はほぼない。
- **「ハッシュテーブルより遅い」→ 用途が違う**。ハッシュは O(1) だが範囲検索不可。BST は範囲・順序クエリが得意。
- **AVL木と赤黒木の選択**：ほとんどの標準ライブラリは赤黒木採用。AVL を自分で実装する必要はほぼない。
- **削除が複雑**：後継ノード（右部分木の最小値）との置き換えが必要。実装バグが起きやすい箇所。

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のインデックス**：`CREATE INDEX` で作成される B-Tree インデックスは BST の多分木版。`EXPLAIN ANALYZE` で "Index Scan" が出ていれば BST 系の恩恵を受けている。

- **範囲検索の最適化**：`WHERE created_at BETWEEN '2026-01-01' AND '2026-07-29'` のような日付範囲クエリは B-Tree インデックスが有効。ハッシュインデックスでは不可。

- **ソート済み結果の取得**：`ORDER BY` + インデックスがある場合、PostgreSQL は Index Scan で追加ソート不要で返せる（ソート済みで格納されているため）。

- **Cloud Run での注意**：インメモリのソートコンテナが必要な場合は `sortedcontainers` を使う。コールドスタート時の初期化コストは O(n log n)。

```sql
-- B-Treeインデックスが有効な範囲クエリ
CREATE INDEX idx_posts_created ON posts(created_at);
EXPLAIN ANALYZE SELECT * FROM posts
WHERE created_at >= '2026-07-01' ORDER BY created_at;
-- → Index Scan が出れば OK
```
