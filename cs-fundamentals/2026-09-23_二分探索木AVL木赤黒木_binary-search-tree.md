# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序付きツリー。検索・挿入・削除をO(log n)で行えるが、偏ると最悪O(n)になる。
AVL木・赤黒木は「自己平衡型BST」で、常にO(log n)を保証する。
DBインデックス（B-Tree）やOS・言語標準ライブラリの順序付きMap（`std::map`、Pythonの`SortedList`相当）の基礎となる。

---

## 仕組みの要点

### 二分探索木（BST）
- 各ノードは左サブツリー < 自分 < 右サブツリー
- 検索：根から比較して左右に降りる
- 挿入：検索で到達したNULL位置に追加
- **問題点**：昇順データを順に挿入すると一直線になりO(n)に退化

```
  5          1
 / \    →    \  ← 退化した木
3   7          2
              \
               3
```

### AVL木
- 各ノードで「左右サブツリーの高さの差（バランス因子）」を常に±1以内に保つ
- 挿入・削除後に違反があれば**回転（rotation）**で修正
- 回転の種類：右回転、左回転、左右回転、右左回転
- 高さ：最大 1.44 × log₂(n)

### 赤黒木
- 各ノードに「赤 or 黒」の色を持たせる
- 5つのルールを維持して高さをO(log n)に保つ
  1. ノードは赤か黒
  2. 根は黒
  3. 赤ノードの子は必ず黒（赤が連続しない）
  4. 全ての葉（NIL）は黒
  5. 任意のノードからその子孫の葉までの「黒ノード数」は等しい
- AVL木より若干高さが高いが、**回転回数が少なく挿入・削除が高速**

---

## 計算量

| 操作 | BST（最良/最悪）| AVL木 | 赤黒木 |
|------|--------------|-------|--------|
| 検索 | O(log n) / O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) / O(n) | O(log n)※回転多め | O(log n)※回転少め |
| 削除 | O(log n) / O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) |

- **AVL木** → 読み取り頻度が高い場合に有利（より厳密にバランス）
- **赤黒木** → 挿入・削除が多い場合に有利（Linuxカーネル、Java TreeMapで採用）

---

## コード例（Python）

```python
class BSTNode:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

def insert(root, val):
    if not root:
        return BSTNode(val)
    if val < root.val:
        root.left = insert(root.left, val)
    elif val > root.val:
        root.right = insert(root.right, val)
    return root

def search(root, val):
    if not root or root.val == val:
        return root
    if val < root.val:
        return search(root.left, val)
    return search(root.right, val)
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**：ランダムデータならほぼO(log n)だが、ソート済みデータでO(n)に退化する
- **AVL木と赤黒木の使い分け**：「どちらもO(log n)だから同じ」ではなく、定数係数（回転コスト）が違う
- **Pythonの`dict`はBSTではない**：ハッシュテーブルなのでキー順の反復には使えない。順序付きが必要なら`sortedcontainers.SortedDict`を使う
- **B-TreeはBSTではない**：ノードが複数キーを持ち、ディスクI/O効率のために設計された別物

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連：**

- **Neon（PostgreSQL）のインデックス**：`CREATE INDEX`はB-Tree（赤黒木を多段化したもの）。WHERE句の範囲検索（`BETWEEN`、`>`）はハッシュインデックスではなくB-Treeが必要
- **順序付きデータのメモリ管理**：APIのリアルタイムランキングをRedis SortedSet（スキップリスト実装）で持つ場合、BSTと同等のO(log n)保証を理解しておくと設計判断できる
- **アルゴリズム問題対策**：LeetCodeのMedium以上では平衡木の挙動を前提とした問題が多い。Pythonでは`sortedcontainers`が赤黒木相当
