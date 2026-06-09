# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要
二分探索木（BST）は「左 < 親 < 右」の制約を持つ木構造で、検索・挿入・削除をO(log n)で行える。
ただし偏りが生じると最悪O(n)に劣化するため、**自己平衡木**（AVL木・赤黒木）が実用される。
PostgreSQLのB-TreeインデックスはBSTの多分木拡張であり、DBエンジニアにとって必須の基礎知識。

## 仕組みの要点

### 二分探索木（BST）
- 各ノードで「左部分木 ≤ 自分 ≤ 右部分木」の制約を維持
- 検索: 根から比較しながら左右を選ぶ → O(h)、h=木の高さ
- 問題: ソート済みデータを挿入すると h=n の線形リストに劣化

### AVL木
- 各ノードで左右部分木の高さ差（平衡係数）≦1 を保証
- 挿入・削除後に**回転操作**（左回転・右回転・二重回転）で修正
- 高さ保証: h ≤ 1.44 × log₂(n) → 検索O(log n)を厳密に保証
- 回転頻度が高いため書き込みコストがやや高い。読み取り中心の用途向き

### 赤黒木
- 各ノードが赤/黒の色を持つ。制約: ルートは黒、赤ノード連続禁止、全パスの黒ノード数が等しい
- 高さ保証: h ≤ 2 × log₂(n+1)（AVL木より緩め）
- 回転・色変更が少なく挿入・削除が速い
- 多くの言語の標準実装で採用（JavaのTreeMap、C++のstd::mapなど）

## 計算量・パフォーマンス特性

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n) | O(n) |

読み取り多 → AVL木、書き込み多 → 赤黒木が有利な傾向。

## コード例（Python: 基本BST）

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
            if not node: return Node(v)
            if v < node.val: node.left = _ins(node.left, v)
            elif v > node.val: node.right = _ins(node.right, v)
            return node
        self.root = _ins(self.root, val)

    def search(self, val):
        node = self.root
        while node:
            if val == node.val: return True
            node = node.left if val < node.val else node.right
        return False
```

Pythonで自己平衡木が必要な場合は `sortedcontainers.SortedList` を使う（内部はリスト分割で近似実装）。

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ソート済み・逆順データ挿入でO(n)に劣化。本番では必ず自己平衡木を使う
- **AVL木 > 赤黒木ではない**: 読み取り中心ならAVL、書き込みが多いなら赤黒木が速い
- **削除の実装が複雑**: 中順後継ノード（in-order successor）で置換する。実装バグの温床になりやすい
- **Pythonの`dict`はBSTではない**: ハッシュテーブルなので順序保証なし。範囲検索が必要なら別途ツールが必要

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **NeonのB-Treeインデックス**: PostgreSQLのインデックスはBSTの多分木拡張。`WHERE id = ?` や `BETWEEN` の速さはこれが理由
- **ORDER BY + インデックス**: ソート済みBSTを使うため、インデックスがある列のORDER BYはO(n)スキャンを回避できる
- **Cloud Runのインメモリ管理**: レート制限のタイムウィンドウやスコアランキングをメモリ上で管理する場面で自己平衡木が有用
- **FastAPIの依存注入キャッシュ**: ルートのソート済み管理など、挿入と範囲検索が混在する場面で検討する価値がある
