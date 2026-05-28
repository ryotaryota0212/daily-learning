# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

「探索・挿入・削除をすべてO(log n)で実現する」自己平衡木は、DBインデックスやOSのスケジューラー・ルーティングテーブルに広く使われる。単純な二分探索木（BST）は最悪O(n)に劣化するため、実用ではAVL木か赤黒木が選ばれる。データベースの実行計画やインデックス設計を理解する土台になる。

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは「左子 < 親 ≤ 右子」の不変条件を保つ
- 探索：根から大小比較で左右に降りるだけ
- **問題**：挿入順序が偏ると木が線形リスト化し O(n) に劣化

```
挿入順序 1,2,3,4,5 → 右に伸びるだけの偏った木
    1
     \
      2
       \
        3
```

### AVL木
- 任意のノードで `|左の高さ − 右の高さ| ≤ 1` を常に保証
- 挿入・削除後に**回転**（左回転・右回転・ダブル回転）で再平衡
- 高さは常に ≤ 1.44 log n → 探索が高速
- 回転コストが高いため、更新頻度が高いと赤黒木より遅い

### 赤黒木
- 各ノードを赤か黒に着色し、以下の規則を守る：
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 根から葉（NIL）までの黒ノード数は全パスで一定（黒高さ）
- AVL木より平衡条件が緩い → **回転回数が最大2回**で済む
- 高さは ≤ 2 log(n+1)
- C++ STL の `map`/`set`、Java の `TreeMap`、Linux カーネルのCFSスケジューラーで採用

## 計算量・パフォーマンス特性

| 操作 | BST平均 | BST最悪 | AVL木 | 赤黒木 |
|------|---------|---------|-------|--------|
| 探索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 回転回数（挿入） | − | − | O(log n) | O(1) 償却 |

- **読み取り重視** → AVL木（高さが低い分だけ探索が速い）
- **書き込み重視** → 赤黒木（回転が少ない）

## コード例（BST の基本操作）

```python
class Node:
    def __init__(self, val):
        self.val, self.left, self.right = val, None, None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, v):
            if not node:
                return Node(v)
            if v < node.val:
                node.left = _ins(node.left, v)
            else:
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

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**：ソート済みデータを順番に挿入すると線形リストになる
- **Pythonに組み込みの平衡木はない**：`dict`/`set`はハッシュテーブル。順序付きが必要なら `sortedcontainers.SortedList`（実装はB-treeに近い）
- **削除は挿入より複雑**：子が2つあるノードは「中順後継者（右部分木の最小値）」と値を入れ替えてから削除
- **AVL木のダブル回転**：左右不均衡（LR/RL）の場合は2回の回転が必要。1回で済む場合と混同しやすい

## 実務での使いどころ

- **Neon（PostgreSQL）のインデックス**：`CREATE INDEX` で作られるB-Treeは赤黒木を多分木に拡張したもの。`WHERE id BETWEEN 100 AND 200` の範囲検索が速い理由はBSTの性質そのもの
- **実行計画の読み方**：`EXPLAIN` で `Index Scan` が出るのは木構造を降りてO(log n)で探索している証拠
- **FastAPI でのソート済みデータ管理**：ランキングや時系列データを順序付きで保持したい場合、`SortedList` を検討
- **Cloud Run**：Linux カーネルのCFSスケジューラーがコンテナのプロセス管理に赤黒木を使用している
