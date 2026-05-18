# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」という順序を保つ木構造で、探索・挿入・削除が平均 O(log n) で行える。
しかし単純な BST は偏り（最悪 O(n) の線形リスト）が生じるため、自己平衡木である AVL 木・赤黒木が実用される。
Python の `sortedcontainers`、Java の `TreeMap`、PostgreSQL のインデックスに赤黒木が使われており、
「なぜインデックスが速いか」「なぜソート済みコレクションが高速か」を理解する基礎になる。

---

## 仕組みの要点

### 二分探索木 (BST)

```
       8
      / \
     3   10
    / \    \
   1   6    14
```

- **探索**: 根から「左へ行くか右へ行くか」を繰り返す → 木の高さ h に依存
- **挿入**: 探索して空きノードに追加
- **削除**: 子なし/子1つ/子2つ（中順後継者で置換）の3ケースがある
- **偏り問題**: 昇順に挿入すると高さ = n の線形木になる

### AVL 木（厳密な平衡）

- 各ノードで「左右の高さの差（平衡因子）≤ 1」を保つ
- 違反したら **回転（左回転・右回転・二重回転）** で修正
- 挿入・削除のたびに平衡チェック → 赤黒木より回転回数が多いが、高さが低い
- **読み込み多め**の用途に有利

### 赤黒木（緩やかな平衡）

- 各ノードを赤/黒に色付け、以下の5規則で近似的平衡を保つ:
  1. 根は黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 根から葉までの黒ノード数はすべて等しい（黒高さ）
- 高さ保証: `2 * log(n+1)` 以下（AVL より若干高いが回転が少ない）
- **書き込み多め**の用途に有利（Linux カーネル、Java `TreeMap` で採用）

---

## 計算量まとめ

| 操作     | BST（平均） | BST（最悪） | AVL 木   | 赤黒木   |
|----------|-------------|-------------|----------|----------|
| 探索     | O(log n)    | O(n)        | O(log n) | O(log n) |
| 挿入     | O(log n)    | O(n)        | O(log n) | O(log n) |
| 削除     | O(log n)    | O(n)        | O(log n) | O(log n) |
| 空間     | O(n)        | O(n)        | O(n)     | O(n)     |

---

## コード例（Python）

```python
class BSTNode:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _ins(node, v):
            if node is None:
                return BSTNode(v)
            if v < node.val:
                node.left = _ins(node.left, v)
            elif v > node.val:
                node.right = _ins(node.right, v)
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

- **「BST は常に O(log n)」は誤り** — ランダム入力なら平均 O(log n) だが、ソート済みデータで O(n) になる
- **AVL = 常に最速ではない** — 回転コストが高く、書き込みが多い場合は赤黒木が勝る
- **赤黒木の規則を「丸暗記」しない** — 「黒ノード数が均等 → 高さが log(n) に収まる」という直感を持つ
- **in-order traversal (左→根→右) で昇順ソート** — BSTのこの性質を忘れると実装でハマる

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

| 場面 | 関連 |
|------|------|
| **Neon (PostgreSQL) のインデックス** | B-Tree インデックスは B+ 木（赤黒木の発展形）。`ORDER BY` や範囲検索が速い理由はここ |
| **FastAPI での範囲クエリ最適化** | `WHERE created_at BETWEEN ...` が遅い場合、インデックスの有無を `EXPLAIN ANALYZE` で確認 |
| **ソート済みコレクション** | Python の `sortedcontainers.SortedList` は内部的に平衡木を使用。ランキング機能などに活用 |
| **Cloud Run のルーティング** | API ゲートウェイのルートツリーもトライ木（BST の派生）で O(log n) 探索 |

### 実務で押さえるポイント

- DB インデックスを貼るカラムを選ぶ根拠が「BST の探索コスト削減」にある
- `SortedList` を使うと挿入しながら中央値・パーセンタイルを O(log n) で計算できる
- 赤黒木の「概念」を知ることで、標準ライブラリの計算量保証を正しく読めるようになる
