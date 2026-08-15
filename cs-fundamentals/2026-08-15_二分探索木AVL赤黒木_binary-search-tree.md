# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、探索・挿入・削除を効率的に行える基盤データ構造。  
しかし素朴なBSTは入力順によって偏り（最悪O(n)）が生じるため、AVL木・赤黒木のような**自己平衡木**が実務で使われる。  
DBのB-Treeインデックス、Pythonの`sortedcontainers`、JavaのTreeMapなど、順序付きデータが絡む場所で常に裏方として機能する。

---

## 仕組みの要点

### 二分探索木（BST）

- ノードは `(値, 左子, 右子)` を持つ
- 探索：根から「左か右か」を繰り返すだけ
- 挿入：探索で終端（None）まで下り、そこに追加
- 削除：子が2つある場合は「中順後継（右部分木の最小値）」と置換
- **問題点**：昇順で挿入すると一本道になりO(n)に劣化

```
  5              5
 / \              \
3   7   →(8挿)→   7
                    \
                     8   ← 偏りが続くと線形リストに
```

### AVL木（高さ平衡）

- 各ノードに**平衡因子 = 左高さ - 右高さ**を保持
- 絶対値が2以上になったら**回転（Rotation）**で修正
  - 左回転 / 右回転 / 左右回転 / 右左回転 の4パターン
- 高さが常に O(log n) に保たれる（厳密平衡）
- 挿入・削除のたびに回転が発生 → 書き込みコストが高め

### 赤黒木（色による弱平衡）

- 各ノードを**赤か黒**に色付けし、以下の5ルールを維持：
  1. 全ノードは赤か黒
  2. 根は黒
  3. 赤ノードの子は必ず黒（赤が連続しない）
  4. 任意のノードから葉までの黒ノード数は等しい（黒高さ一定）
  5. 葉（NILノード）は黒
- 高さは最大 2·log(n+1) — AVL木より少し高くなりうるが許容範囲
- 挿入・削除後は**色の変更（Recolor）+ 最小限の回転**で修正
- AVL木より回転が少ない → 書き込みが多いケースで有利

---

## 計算量・パフォーマンス特性

| 操作       | BST（平均） | BST（最悪） | AVL木      | 赤黒木     |
|------------|------------|------------|------------|------------|
| 探索       | O(log n)   | O(n)       | O(log n)   | O(log n)   |
| 挿入       | O(log n)   | O(n)       | O(log n)   | O(log n)   |
| 削除       | O(log n)   | O(n)       | O(log n)   | O(log n)   |
| 空間       | O(n)       | O(n)       | O(n)       | O(n)       |
| 回転回数   | —          | —          | 最大O(log n) | 最大O(1)  |

- **探索頻度 >> 更新頻度** → AVL木（より厳密な高さ保証）
- **更新頻度が高い** → 赤黒木（回転が少ない）
- Linuxカーネルのスケジューラ（CFS）・JavaのTreeMap・C++のstd::mapは赤黒木を採用

---

## コード例（Python — 素朴なBSTの探索・挿入）

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
            if node is None:
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

---

## よくある誤解・落とし穴

- **「BSTなら常にO(log n)」は誤り** — ソート済みデータを挿入するとO(n)に劣化する。自己平衡木が必要。
- **AVL木は回転が重い** — 挿入のたびに祖先を辿って平衡因子を更新するため、更新が多いワークロードでは赤黒木が速い。
- **削除が最も複雑** — 特にBSTで子が2つあるノードの削除は中順後継の付け替えが必要。AVL/赤黒でも最大3回の回転が走る。
- **Pythonにはネイティブの平衡BST実装がない** — `sortedcontainers.SortedList` はスキップリスト/リストの分割で近似。本物のBSTが必要なら`bintrees`や自前実装。

---

## 実務での使いどころ（FastAPI + Neon + Cloud Runスタック）

- **Neon（PostgreSQL）のB-Tree インデックス**はBSTを多分木に拡張した構造。`CREATE INDEX`の計算量の直感（O(log n)探索）はBSTの知識がそのまま活きる。
- **範囲クエリ（BETWEEN, ORDER BY）**が速い理由は木の中順走査がソート順を自然に返すから。
- **FastAPIのルーティング**は辞書（ハッシュ）ベースだが、数値・日付での範囲検索API設計時にDBインデックスとの相性を意識する際にBSTの知識が判断の根拠になる。
- 大量データのメモリ内ランキング・順位計算が必要になった場合、`SortedList`（平衡木的構造）vs `heapq`（ヒープ）の選択にこの知識が効く。
