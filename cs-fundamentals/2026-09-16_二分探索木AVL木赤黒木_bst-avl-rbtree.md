# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という順序制約を持つ木構造で、検索・挿入・削除をO(log n)で実現する。
しかしナイーブなBSTは偏りが生じるとO(n)に劣化するため、自動平衡する**AVL木**や**赤黒木**が実用上使われる。
RDBMSのインデックス（B-Tree）、JavaのTreeMap、C++のstd::mapなど、あらゆる順序付きデータ構造の基礎となる。

---

## 仕組みの要点

### 二分探索木（BST）の基本

- **構造**：各ノードは左部分木（すべて親より小さい）と右部分木（すべて親より大きい）を持つ
- **中順巡回**（左→親→右）で要素が昇順に並ぶ
- 問題：挿入順によっては一直線になり、最悪O(n)に劣化

```
挿入順: 5, 3, 7, 2, 4  →  バランスが良い
挿入順: 1, 2, 3, 4, 5  →  右に偏った線形リスト（最悪ケース）
```

### AVL木の平衡維持

- 各ノードで「左右の部分木の高さの差（平衡因子）が±1以内」を保証
- 挿入・削除後に**回転操作**（左回転・右回転・二重回転）で平衡を回復
- 高さが常にO(log n)に保たれる
- 検索は赤黒木より高速（より厳密に平衡）だが、回転が多く書き込みコストがやや高い

### 赤黒木の平衡維持

- 各ノードに「赤」か「黒」の色を付け、以下の5条件を維持：
  1. 全ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードからすべての葉までの黒ノード数は同一
- AVL木より緩い条件だが、高さはO(2 log n)以内に保証
- 回転・再彩色コストがAVL木より低く、**挿入・削除が多いユースケース**に向く

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- **AVL木**：検索が多い場合に有利（高さが最小）
- **赤黒木**：挿入・削除が多い場合に有利（回転回数が少ない）
- 実用では赤黒木が多く採用される（Linux kernel、Java TreeMap、C++ std::map）

---

## コード例（Python: ナイーブBSTの検索・挿入）

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
            if node is None:
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

- **「BSTは常にO(log n)」は誤り**：平衡が崩れると最悪O(n)。ソート済みデータを順番に挿入すると一直線になる
- **「AVL木と赤黒木は同じ」は誤り**：AVL木は最大高さが1.44 log n、赤黒木は2 log n。検索頻度と書き込み頻度で使い分ける
- **削除の難しさ**：BSTの削除は挿入より複雑。削除対象が2つの子を持つ場合、中順後継者（右部分木の最小値）で置き換える
- **同値の扱い**：「等しい場合は左に入れる or 右に入れる」を設計時に決めておかないと不整合が起きる

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **Neon（PostgreSQL）のインデックス**：`CREATE INDEX`で作られるB-Treeインデックスは赤黒木の発展形（複数キーを1ノードに格納）。BSTの原理を理解すると実行計画（`EXPLAIN ANALYZE`）の読み方が身につく
- **範囲クエリの効率**：`WHERE price BETWEEN 100 AND 500`のような範囲クエリが速いのはB-Treeが順序を保持するから。ハッシュインデックスでは範囲検索できない
- **FastAPIのルーティング**：一部のWebフレームワークはルーティングテーブルにトライ木（BSTの応用）を使用。パスパラメータのマッチングに活用
- **インデックス設計のヒント**：カーディナリティが高い列（ユーザーID、日時など）はB-Treeインデックスが効果的。低カーディナリティ（フラグ値など）は効果が薄い
