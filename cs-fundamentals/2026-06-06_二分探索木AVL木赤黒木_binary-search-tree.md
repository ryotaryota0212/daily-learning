# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要
二分探索木（BST）は「左の子 < 親 < 右の子」の不変条件を保つ木構造。
検索・挿入・削除を平均O(log n)で実現できる。
ただし偏った挿入でO(n)に劣化するため、AVL木・赤黒木は**自動的なバランス調整**（回転）でO(log n)を保証する。
PostgreSQLやLinuxカーネルの内部構造、Java TreeMapなど実装至るところで使われる。

---

## 仕組みの要点

### 二分探索木（BST）の基本
- 検索：根から左右を比較して降りる
- 挿入：検索してNULLの場所に追加
- 削除：後継ノード（右部分木の最小値）と置換して削除
- **問題点**：昇順挿入など偏ったデータで深さがO(n)になる

```
挿入: 5, 3, 7, 1, 4 → バランス良好
挿入: 1, 2, 3, 4, 5 → 右に一直線（最悪ケース）
```

### AVL木
- **高さバランス条件**：各ノードで左右の高さの差（バランス因子）が -1, 0, +1 のいずれか
- 挿入・削除後に条件を違反したら**回転**で修正
  - 右回転（Left-Left ケース）
  - 左回転（Right-Right ケース）
  - 左右回転（Left-Right ケース）
  - 右左回転（Right-Left ケース）
- **特徴**：高さが厳密に管理されるため検索が高速。挿入・削除は回転が多め

### 赤黒木
- 各ノードに「赤」「黒」の色を付け、以下の条件を保つ
  1. 根は黒
  2. 赤ノードの子は黒（赤が連続しない）
  3. 全経路の黒ノード数が同じ
- **特徴**：AVL木より条件が緩い → 回転が少なく挿入・削除が速い。高さはAVL木より若干高い
- **実用例**：Java TreeMap、C++ std::map、Linuxカーネルのプロセス管理

---

## 計算量・パフォーマンス特性

| 操作     | BST平均  | BST最悪  | AVL木    | 赤黒木   |
|--------|--------|--------|--------|--------|
| 検索     | O(log n) | O(n)   | O(log n) | O(log n) |
| 挿入     | O(log n) | O(n)   | O(log n) | O(log n) |
| 削除     | O(log n) | O(n)   | O(log n) | O(log n) |
| 空間     | O(n)   | O(n)   | O(n)   | O(n)   |

- **AVL木 vs 赤黒木**：検索が多いならAVL木（より厳密にバランス）、挿入・削除が多いなら赤黒木

---

## コード例（Python — BST 検索・挿入）

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

- **「BSTは常にO(log n)」は誤り**：偏ったデータでO(n)になる。ランダムデータ前提の楽観的な期待値
- **AVL木が最強とは限らない**：回転コストのために書き込み負荷が高いワークロードでは赤黒木が優位
- **削除は複雑**：後継ノードの選び方や回転処理でバグが出やすい。実務では既存ライブラリに任せる
- **ハッシュテーブルとの使い分け**：BSTは「範囲検索」「順序付き列挙」が得意。ハッシュは点検索が得意

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連
- **Neon（PostgreSQL）のB-Treeインデックス**はBSTを多分木に拡張したもの。BSTの理解がインデックス設計の土台になる
- **範囲クエリ**（`WHERE created_at BETWEEN ...`）が速いのはB-Tree（BST系）が順序を保持しているから
- **Cloud Run の水平スケール**：アプリ側でin-memoryのソート済み構造が必要な場合、Pythonの`sortedcontainers.SortedList`（赤黒木ベース）が有効
- **FastAPIのルーティング**：内部的に前置木（Trie）を使うが、BST系の「比較して降りる」考え方は同じ

### 判断指標
- 順序を保ちながら挿入・削除・検索したい → 赤黒木（Python: `sortedcontainers`）
- 範囲クエリ → DBのB-Treeインデックスに任せる
- 単純な点検索だけ → ハッシュテーブルの方が速い
