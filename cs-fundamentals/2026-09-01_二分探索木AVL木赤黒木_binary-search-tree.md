# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という順序制約を持つ木構造。
探索・挿入・削除が平均O(log n)で動作し、ハッシュテーブルと違い「範囲検索」「ソート済み列挙」が得意。
ただし素朴なBSTは入力によって偏り、最悪O(n)になる。実務ではAVL木・赤黒木などの**自己平衡木**が使われ、PostgreSQLのB-Treeインデックスもこの系譜にある。

---

## 仕組みの要点

### 二分探索木（BST）

- 各ノードは `(key, left, right)` を持つ
- 探索：目的値と比較しながら左右に降りる
- 挿入：葉ノードとして追加
- 削除：後継ノード（右部分木の最小値）と入れ替えてから除去
- 問題：昇順データを挿入すると一本道の線形リストになる → O(n)

```
挿入順: 3, 1, 5, 0, 2, 4, 6
        3
       / \
      1   5
     / \ / \
    0  2 4  6   ← バランスが取れた状態（たまたま）
```

### AVL木

- 各ノードに**高さ（height）**を管理する
- 左右の高さの差（バランス係数）が常に -1, 0, +1 を保つ
- 制約違反時に**回転（rotation）**で修正
  - 右回転・左回転・左右回転・右左回転の4種
- 挿入・削除のたびに最大O(log n)回の回転が発生

### 赤黒木

- 各ノードに**色（赤 or 黒）**を付ける
- 制約：
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤連続禁止）
  3. すべての葉からNILまでの黒ノード数が等しい
- これにより高さが 2log(n+1) 以内に収まる
- AVL木より少し緩い制約 → **挿入・削除の回転数が少ない**（実装が速い）
- C++の`std::map`、Javaの`TreeMap`、Linuxカーネルのスケジューラに採用

---

## 計算量・パフォーマンス特性

| 操作 | 素朴BST（平均） | 素朴BST（最悪） | AVL木 | 赤黒木 |
|------|----------------|----------------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 範囲探索 | O(log n + k) | O(n) | O(log n + k) | O(log n + k) |

- AVL木はより厳密に平衡 → **探索が高速**（読み取り多いケース）
- 赤黒木は挿入・削除の回転が少ない → **書き込み多いケース**向き
- kは取得件数

---

## コード例（Python: 素朴BST）

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None

class BST:
    def __init__(self): self.root = None

    def insert(self, key):
        def _insert(node, k):
            if node is None: return Node(k)
            if k < node.key: node.left = _insert(node.left, k)
            elif k > node.key: node.right = _insert(node.right, k)
            return node
        self.root = _insert(self.root, key)

    def search(self, key):
        node = self.root
        while node:
            if key == node.key: return True
            node = node.left if key < node.key else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」** → ×。偏った入力では最悪O(n)。本番には必ず平衡木を使う
- **「AVL木の方が常に速い」** → ×。書き込みが多い場面では赤黒木の方が有利
- **「Pythonに標準の平衡木がない」** → そのとおり。`sortedcontainers.SortedList`（外部ライブラリ）か`heapq`で代替
- **削除の実装が難しい** → 後継ノード（in-order successor）の探し方を間違えやすい

---

## 実務での使いどころ

**FastAPI + Neon（PostgreSQL） + Cloud Run スタックとの関連**

- **PostgreSQLのB-Treeインデックス**：赤黒木の派生。`CREATE INDEX`で自動生成される
  - `EXPLAIN ANALYZE`でIndex Scanが出るとき、内部でB-Treeを辿っている
  - 等値検索だけでなく`BETWEEN`や`ORDER BY`でもインデックスが効くのは木構造だから
- **範囲クエリの最適化**：`WHERE created_at BETWEEN '2026-01-01' AND '2026-09-01'`は木の範囲探索と対応
- **ハッシュとの使い分け**：等値検索のみならHashインデックスの方が速いが、ソートや範囲が必要ならB-Tree一択
- `sortedcontainers.SortedList`：FastAPIのリクエスト内でソート済みリストへのO(log n)挿入が必要な場面（例：リアルタイムランキング）で有効
