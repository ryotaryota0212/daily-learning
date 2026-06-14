# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要
二分探索木（BST）は「左 < 親 < 右」の順序付きツリー構造。検索・挿入・削除がO(log n)になる理想があるが、偏りが生じるとO(n)に劣化する。AVL木・赤黒木は「自己平衡型BST」で常にO(log n)を保証する。DBインデックスの理解や、言語標準ライブラリのMap/Set内部実装の把握に直結する。

## 仕組みの要点

### 二分探索木（BST）
- 各ノード: 左サブツリー < ノード値 < 右サブツリーを保証
- 検索: 値と比較しながら左右に降りるだけ
- 挿入: 検索で辿り着いたNullスロットに追加
- **問題**: ソート済みデータを順に挿入 → リスト状になりO(n)に劣化

### AVL木
- BSTに「左右サブツリーの高さの差が1以下（平衡因子）」を追加
- 差が2以上になったら**回転（rotation）**で再構成: LL / RR / LR / RL の4種
- 常に高さをO(log n)に保つ → 検索が高速
- 回転コストが高いため挿入・削除の多いユースケースでは赤黒木に劣る

### 赤黒木（Red-Black Tree）
- 各ノードに「赤」か「黒」の色を付けたBST
- 5規則（ルートは黒、赤の子は黒、任意ノードから葉まで黒ノード数が同じ等）でバランス保証
- AVLより緩いバランス → 回転回数が少なく挿入・削除が速い
- 実例: Linux カーネルのCFSスケジューラ、C++ `std::map`、Java `TreeMap`

## 計算量・パフォーマンス特性

| 操作     | BST(平均)  | BST(最悪) | AVL木    | 赤黒木   |
|--------|----------|---------|--------|--------|
| 検索     | O(log n) | O(n)    | O(log n) | O(log n) |
| 挿入     | O(log n) | O(n)    | O(log n) | O(log n) |
| 削除     | O(log n) | O(n)    | O(log n) | O(log n) |
| 挿入時回転 | -        | -       | 多い     | 最大2回  |

- 検索が主ならAVL木（より厳密なバランス）
- 挿入・削除が多いなら赤黒木（回転コスト低）

## コード例（Python - BSTの挿入と検索）

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

## よくある誤解・落とし穴

- **「BSTなら常にO(log n)」→ 誤り**: ソート済み入力などで偏ると最悪O(n)
- **自前実装しようとする**: 実務では `sortedcontainers.SortedList` (Python) や言語組み込みMapで十分
- **削除の複雑さを軽視**: 削除は最も実装が難しい（後継ノードへの置き換えが必要）
- **AVLと赤黒木を混同**: どちらもO(log n)だが定数係数と適用場面が異なる

## 実務での使いどころ

- **Neon（PostgreSQL）のインデックス**: B-Tree はBSTの多分木拡張。範囲検索が速い理由の理解につながる
- **FastAPI**: レスポンスのソート済みデータ管理に `sortedcontainers.SortedList` が有用
- **Cloud Run / Linux**: カーネルの仮想メモリ管理（vm_area_struct）や CFS スケジューラが赤黒木を使用
- 「同じO(log n)でも実装によって定数倍が違う」という感覚がDBインデックス選択・チューニングの判断力につながる
