# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は検索・挿入・削除を効率化するデータ構造。実装は単純だが、偏りが生じると最悪 O(n) に劣化する。AVL木・赤黒木は自動バランス機構を持ち、常に O(log n) を保証する。データベースインデックス（PostgreSQL の B-Tree）や言語標準ライブラリ（Java の `TreeMap`、C++ の `std::map`）の根幹技術であり、「なぜインデックスは速いのか」を理解する上で必須の知識。

## 仕組みの要点

### 二分探索木（BST）

- 各ノードの左部分木 ≤ ノード値 ≤ 右部分木という不変条件を維持
- 中順走査（in-order traversal）で昇順ソート済みシーケンスが得られる
- 連続した昇順データを順挿入すると一直線になり O(n) に劣化（最悪ケース）

### AVL木

- 各ノードで左右部分木の高さの差（平衡因子）が常に -1, 0, +1 になるよう維持
- 挿入・削除後に「回転（rotation）」操作でバランスを回復（右回転・左回転・二重回転）
- 厳密なバランス → 赤黒木より検索が高速だが、回転コストがやや高い

### 赤黒木

- 各ノードに赤・黒の色を付与、5つの制約で「弱いバランス」を保証
- 制約: ルートは黒 / NULL葉は黒 / 赤ノードの子は黒 / どの経路も同じ黒ノード数
- AVL木より回転数が少なく、挿入・削除が高速 → Linux カーネル・Java `TreeMap`・C++ `std::map` で採用

## 計算量・パフォーマンス特性

| 操作 | BST (平均) | BST (最悪) | AVL木 | 赤黒木 |
|------|-----------|-----------|-------|--------|
| 検索 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 挿入 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 削除 | O(log n)  | O(n)      | O(log n) | O(log n) |
| 回転 | —         | —         | 最大 2回 | 最大 3回 |
| 空間 | O(n)      | O(n)      | O(n) | O(n) |

## コード例（Python: BST 基本実装）

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
        return self.search(root.left, val) if val < root.val else self.search(root.right, val)
```

## よくある誤解・落とし穴

- **「BST は常に O(log n)」は誤り** — ランダムデータなら平均 O(log n) だが、ソート済みデータを順挿入すると O(n) になる。実運用で素の BST を使うのは危険
- **AVL木 = 常に高速ではない** — 回転コストが高く、書き込みが多い用途では赤黒木に劣ることがある
- **Python に組み込み BST はない** — `sortedcontainers.SortedList` か `bisect` モジュールを使う。`set`/`dict` は内部的にハッシュテーブルであり BST ではない
- **削除操作は複雑** — 子が2つあるノードの削除では「中順後継ノード」に値を置換する処理が必要

## 実務での使いどころ（FastAPI + Neon + Cloud Run スタック）

- **Neon (PostgreSQL) のインデックス**: `EXPLAIN` で `Index Scan` が出るのは内部が B-Tree（BST の多分木版）。範囲クエリ（`WHERE id BETWEEN 100 AND 200`）が速いのは B-Tree の中順走査で連続した値を効率的に読めるから
- **複合インデックスの設計**: B-Tree の左前一致制約を理解するには BST の「左部分木に小さい値が集まる」原理が土台になる。`(user_id, created_at)` インデックスは `user_id` のみでも使えるが `created_at` 単独では使えない
- **ソート済み範囲クエリ**: `ORDER BY id LIMIT 20` が高速なのは B-Tree の葉ノードがソート済み連結リストで繋がっているから（シーケンシャルアクセス可能）
- **Cloud Run + キャッシュ**: インメモリでソート済みデータを保持するなら `sortedcontainers.SortedList` が有用。挿入・検索が O(log n) で、レートリミットのウィンドウ管理などに使える
