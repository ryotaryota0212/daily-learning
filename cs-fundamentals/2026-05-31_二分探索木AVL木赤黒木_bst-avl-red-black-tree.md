# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の性質を持つ木構造で、探索・挿入・削除を効率化する。
しかし素朴なBSTは偏りが生じて最悪O(n)になる。この問題を解決するのが自己平衡木（AVL木・赤黒木）。
実務ではDBのインデックス、言語標準ライブラリの`set`/`map`が内部でこれらを使っている。

---

## 仕組みの要点

### 二分探索木（BST）

```
       8
      / \
     3   10
    / \    \
   1   6    14
      / \
     4   7
```

- **探索**: ルートから「小さければ左、大きければ右」を繰り返す
- **挿入**: 探索で到達した葉の位置に追加
- **削除**: 3パターン（葉・子1つ・子2つ）→子2つの場合は「中順後継者」で置換
- **問題点**: 昇順挿入 `[1,2,3,4,5]` → 右に伸びる線形リスト → O(n)

### AVL木（高さ平衡木）

- 各ノードで `|左高さ - 右高さ| ≤ 1` を常に維持
- 挿入・削除後に**回転（rotation）**で再バランス
- 回転の種類: LL回転・RR回転・LR回転・RL回転
- **高さ保証**: 常に O(log n)
- **欠点**: 回転頻度が高く書き込みコストが大きい

### 赤黒木

- 各ノードに赤/黒の色を付け、以下の規則でバランスを保つ
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意のノードから葉までの黒ノード数が等しい（黒高さ一定）
- AVL木より**緩い平衡条件** → 回転回数が少ない
- **高さ保証**: 最大 2×log(n+1)
- C++の`std::map`、JavaのTreeMap、Linuxカーネルのスケジューラで採用

---

## 計算量

| 操作       | BST（平均） | BST（最悪） | AVL木    | 赤黒木   |
|------------|-------------|-------------|----------|----------|
| 探索       | O(log n)    | O(n)        | O(log n) | O(log n) |
| 挿入       | O(log n)    | O(n)        | O(log n) | O(log n) |
| 削除       | O(log n)    | O(n)        | O(log n) | O(log n) |
| 空間       | O(n)        | O(n)        | O(n)     | O(n)     |

**AVL木 vs 赤黒木**:
- 検索が多い → AVL木（高さが低くキャッシュに有利）
- 挿入・削除が多い → 赤黒木（回転が少ない）

---

## コード例（Python: 素朴なBST）

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

> Python標準ライブラリにBSTはない。`sortedcontainers.SortedList`（内部はB-tree的構造）を使うと実用的。

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ソート済みデータを挿入すると線形に劣化する
- **AVL木の回転を全部覚える必要はない**: 実務では既存ライブラリを使う。重要なのは「なぜ回転が必要か」の直感
- **赤黒木は完全平衡ではない**: AVL木より高さが高くなる場合がある（最大2倍程度）
- **削除が最も複雑**: 赤黒木の削除は挿入より場合分けが多い（実装は参照実装を使うべき）

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **PostgreSQL（Neon）のインデックス**: B-Treeインデックスは赤黒木の考え方を多次元化したもの。`CREATE INDEX`の仕組みを理解する出発点になる
- **APIの範囲検索**: 「price BETWEEN 100 AND 500」はBSTの性質（中順探索）で効率化されている
- **FastAPIのルーティング**: トライ木（Trie）はBSTの応用。パスパラメータのマッチングに関係する
- **Cloud Runのスケーリング判断**: OS内部のスケジューラが赤黒木でタスク管理（Linux CFS）。コンテナのCPUスロットリング理解に繋がる

### 面接・競プロでの頻出パターン

- k番目に小さい要素を探す → BST + 部分木サイズ
- 動的に中央値を求める → ヒープ2本 or 平衡BST
- 区間の重なり検出 → 拡張BST（interval tree）
