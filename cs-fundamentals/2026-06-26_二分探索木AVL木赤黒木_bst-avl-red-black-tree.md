# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は検索・挿入・削除をO(log n)で行える基本的なデータ構造。ただし偏ったデータではO(n)に劣化する。AVL木・赤黒木はこの弱点を「自己平衡」で補い、最悪計算量をO(log n)に保証する。実務ではDB索引（B-Tree）、言語標準ライブラリのOrderedMap/Set、Linuxスケジューラの内部実装に使われており、「なぜインデックスが速いか」を理解する土台になる。

---

## 仕組みの要点

### 二分探索木（BST）
- **不変条件**: 左部分木のすべての値 < ノード値 < 右部分木のすべての値
- 検索・挿入・削除はすべて**O(h)**（h = 木の高さ）
- ランダムデータ → h ≈ log n（効率的）
- 昇順・降順データを挿入 → 木が一直線になりh = n（最悪ケース）

### AVL木
- **平衡条件**: 各ノードで `|左高さ - 右高さ| ≤ 1` を常に維持
- 挿入・削除後に違反箇所を特定し「ローテーション」で修正
- ローテーションは4種類: LL・RR（1回転）、LR・RL（2回転）
- 高さは常に **≤ 1.44 log n** → 検索はBSTより速い
- 挿入・削除のたびに複数ローテーションが発生しうる（書き込みコストが高め）

### 赤黒木（Red-Black Tree）
- ノードに赤/黒の色を付け、5つの性質で緩やかな平衡を実現:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 葉（NILノード）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意の葉からルートまでの「黒ノード数」は等しい
- これにより高さ ≤ 2 log(n+1) が保証される
- 修正ローテーションがAVL木より少ない → **書き込みが多い場面で有利**
- 採用例: JavaのTreeMap・TreeSet、C++ STLのmap/set、Linuxのepoll

---

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

- **AVL木**: 読み取りが多い場合に有利（より厳密に平衡）
- **赤黒木**: 挿入・削除が多い場合に有利（ローテーション回数が少ない）

---

## コード例（Python: BSTの基本実装）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, val):
            if not node:
                return Node(val)
            if val < node.val:
                node.left = _ins(node.left, val)
            elif val > node.val:
                node.right = _ins(node.right, val)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ソート済みデータを順に挿入するとリストと同じになる。実際のDBが生のBSTでなくB-Tree/赤黒木を採用している理由
- **削除の実装が難しい**: 子が2つあるノードの削除は「中順後継者（右部分木の最小値）」で置換→初実装でよくバグが出る
- **AVL木の実装コスト**: ローテーション4種類の実装はミスが入りやすい。実務では既存ライブラリを使う
- **赤黒木の「緩さ」**: AVL木より最大1.4倍高くなる可能性があるが、それでも保証はO(log n)
- **Pythonに組み込みBSTはない**: `dict`や`set`はハッシュテーブル。範囲検索が必要なら`sortedcontainers.SortedList`を使う

---

## 実務での使いどころ（FastAPI + Neon + Cloud Runスタック）

- **Neon (PostgreSQL)**: B-Treeインデックスは赤黒木と同じ「自己平衡木」の考え方を応用。`WHERE id = X`や`ORDER BY`が速いのはこの仕組みによる。クエリの実行計画（`EXPLAIN ANALYZE`）でIndex Scanが選ばれているか確認する習慣をつける
- **FastAPI**: 範囲検索APIを実装する際、インメモリで`sortedcontainers.SortedList`を使えばソート済み挿入・範囲取得がO(log n)。小規模キャッシュや優先度管理に活用できる
- **Cloud Run（タスクスケジューリング）**: ジョブの優先度管理や期限付きタスクキューを実装するとき、平衡BSTまたはヒープが候補になる。実行時間でソートされたジョブ一覧への次ジョブ検索はO(log n)
