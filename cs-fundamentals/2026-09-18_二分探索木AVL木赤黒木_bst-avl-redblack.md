# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、ソート済みデータの探索・挿入・削除を効率化する。  
ただし素朴なBSTは偏り（最悪O(n)）が起きるため、実用ではAVL木や赤黒木といった**自己平衡BST**が使われる。  
データベースのインデックス（B-Tree）、セット・マップの実装（std::map, Java TreeMap）の根幹にある構造で、「なぜO(log n)が保証できるか」を理解することが重要。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは「左部分木のすべて < 自分 < 右部分木のすべて」を満たす
- 探索：ルートから大小比較して左右に降りるだけ
- 挿入：探索で末端（None）に到達した位置に追加
- **問題**：ソート済みデータを順に挿入すると一直線になり、O(n)に劣化

```
例: 1,2,3,4,5を順に挿入した偏ったBST
1
 \
  2
   \
    3  ← 探索がO(n)になる
```

### AVL木（高さ平衡BST）
- 各ノードで「左右の子の高さの差（バランス因子）≦ 1」を厳守
- 挿入・削除後にバランス因子が崩れたら**回転（Rotation）**で修正
  - 右回転（Right Rotation）：左が重い場合
  - 左回転（Left Rotation）：右が重い場合
  - 左右回転・右左回転：二重回転が必要なケース
- 高さが常にO(log n)に保たれるため、全操作がO(log n)保証
- 厳格な平衡 → 赤黒木より検索が速い、ただし更新コストがやや高い

### 赤黒木（Red-Black Tree）
- 各ノードを「赤」か「黒」に色付けし、以下の制約を維持する
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉（NIL）への経路上の黒ノード数が等しい（黒高さ一定）
- この制約から木の高さが最大 2×log(n+1) に抑えられる
- 回転と色の変更で平衡を修復（AVL木より回転回数が少なく更新が速い）
- **実用例**：Linux カーネルのプロセススケジューラ、Java TreeMap、C++ std::map

---

## 計算量・パフォーマンス特性

| 操作       | 素のBST（最悪） | AVL木    | 赤黒木   |
|-----------|--------------|----------|----------|
| 探索       | O(n)         | O(log n) | O(log n) |
| 挿入       | O(n)         | O(log n) | O(log n) |
| 削除       | O(n)         | O(log n) | O(log n) |
| 平衡保証   | なし          | 厳格     | 緩い     |
| 回転コスト | -            | 挿入最大2回、削除多め | 挿入最大2回、削除最大3回 |

**空間計算量**：O(n)（全ノードで色/高さの付加情報が定数個）

---

## コード例（Python：BST の基本操作）

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

- **「BSTは常にO(log n)」は誤り**：平衡保証がなければ最悪O(n)になる。ランダムデータでは期待値O(log n)だが、ソート済みデータで崩壊する。
- **AVL木と赤黒木の使い分け**：読み取りが多いなら AVL木（厳格な平衡で探索が速い）、更新が多いなら赤黒木（回転が少ない）。
- **削除が一番複雑**：削除には「後継ノード（右部分木の最小値）との交換」が必要で、平衡修復も複雑になる。ライブラリを使うのが現実的。
- **Pythonの`sortedcontainers.SortedList`**：Pythonに標準の平衡BSTはないが、`sortedcontainers`ライブラリがB-Tree的な実装でO(log n)操作を提供する。

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連**

- **PostgreSQL（Neon）のインデックス**：BTreeインデックスは赤黒木の概念を多要素に拡張したB-Tree。`CREATE INDEX`の仕組みを理解するための土台になる。
- **範囲クエリの効率**：`WHERE created_at BETWEEN A AND B`がO(log n)で動くのは、BSTの「中順走査でソート順アクセス」の性質のおかげ。
- **FastAPIのルーティング**：URLルートの内部マッチングにトライ木（Trie）や辞書（ハッシュ）が使われるが、優先度付きルーティングの概念はBST的な順序構造と同じ発想。
- **In-memoryキャッシュの実装**：LRUキャッシュを高度化して「スコア順に取得したい」場合、`SortedList`や平衡BSTが使える（Redis の ZSet もスキップリストで同等の機能を提供）。
