# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という制約を持つ木構造で、探索・挿入・削除を効率的に行える。しかし通常のBSTは偏る（最悪O(n)）ため、**自己平衡木**（AVL木・赤黒木）が実用上重要。PostgreSQLのB-Tree、Javaの`TreeMap`、Pythonの`sortedcontainers`などの内部で活用されている。

## 仕組みの要点

### 二分探索木（BST）

- 探索: ルートから「目標 < 現在なら左、> なら右」を再帰
- 挿入: 探索して空き位置に追加
- 削除: 子が2つある場合、「右部分木の最小値（中順後継者）」と置き換える
- **問題**: ソート済みデータを順に挿入 → 右側に一直線の「偏った木」になりO(n)

### AVL木

- バランス条件: 各ノードで「左右の高さの差（平衡因子）≦ 1」
- 挿入・削除後に平衡因子を確認し、崩れたら**回転操作**で修正
  - 単回転（LL/RR）: 1ノードをルートにする
  - 二重回転（LR/RL）: 2回回転が必要
- 高さが常にO(log n)に保たれる → 探索保証

### 赤黒木

- 各ノードに「赤」「黒」の色を付与し5つの規則で均衡を保つ:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 赤ノードの子は必ず黒（赤が連続しない）
  4. 全パスの黒ノード数が等しい
  5. NULL葉は黒とみなす
- AVL木より厳密さが緩い分、回転が少なく**挿入・削除が高速**
- Linuxカーネルのスケジューラ、Java `TreeMap`、C++ `std::map` に採用

### 比較まとめ

| 特性 | BST | AVL木 | 赤黒木 |
|------|-----|-------|--------|
| 探索 | O(n)最悪 | O(log n) | O(log n) |
| 挿入 | O(n)最悪 | O(log n) | O(log n) |
| 削除 | O(n)最悪 | O(log n) | O(log n) |
| 回転数（挿入） | 0 | 最大2 | 最大2 |
| 回転数（削除） | 0 | O(log n) | 最大3 |
| 向き | — | 探索多い場合 | 挿入・削除多い場合 |

## 計算量・パフォーマンス特性

- **平均ケース**: ランダムデータではBSTも O(log n) だが保証がない
- **AVL木**: 高さ ≦ 1.44 log₂(n+2) — より厳密なバランス
- **赤黒木**: 高さ ≦ 2 log₂(n+1) — AVLより少し高いが回転コスト低い
- **空間**: どちらも O(n)、ただし色/高さのメタデータ1ビット〜数バイト追加

## コード例（Python: BSTの基本操作）

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

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ソート済み挿入でO(n)になる。面接・実務で通常のBSTを自前実装するのは危険
- **AVL木の「回転が多い」は誤解**: 挿入1回あたり最大2回の回転。コストはO(log n)の経路を辿る部分
- **赤黒木は「弱いバランス」ではない**: 最悪高さがO(log n)であることは保証される
- **削除実装の難しさ**: BSTの削除（特に子が2つある場合）は実装ミスが多い。中順後継者の親ポインタ更新を忘れない

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタック

- **NeonのインデックスはB-Tree**: B-Treeは多分木だが原理は赤黒木と同じ（平衡維持）。`CREATE INDEX`の仕組み理解に直結
- **ソート済み範囲クエリ**: `ORDER BY created_at`のような範囲アクセスは木構造が得意 → インデックス有無でコストが劇的に変わる理由を理解できる
- **Pythonの`sortedcontainers.SortedList`**: 赤黒木ベース。FastAPIでリアルタイムランキングや優先度付きキューが必要なら活用
- **自前実装の場面**: 競技プログラミングやシステム設計面接以外では、通常はDBのインデックスや標準ライブラリで代替で十分
