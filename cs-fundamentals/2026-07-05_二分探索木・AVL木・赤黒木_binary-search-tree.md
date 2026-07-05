# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造。検索・挿入・削除をO(log n)で実現できるが、データの挿入順によっては偏りが生じてO(n)に劣化する。AVL木・赤黒木はこの偏りを防ぐ「自己平衡型BST」で、DBのインデックス・標準ライブラリの実装に広く使われる。

## 仕組みの要点

### 二分探索木（BST）

```
       8
      / \
     3   10
    / \    \
   1   6   14
      / \
     4   7
```

- **左部分木**の全ノード < 親ノード < **右部分木**の全ノード
- 検索：根から比較して左右に降りていく
- 偏り問題：昇順で挿入すると右に連なる線形構造になる（O(n)）

### AVL木

- **高さの差（平衡因子）が常に -1, 0, +1** を保つ
- 挿入・削除後に違反ノードを見つけたら回転（rotation）で修正
- 回転の種類：左回転・右回転・左右回転・右左回転の4パターン
- 高さ保証：h ≤ 1.44 log₂(n) → 検索は常にO(log n)
- 赤黒木より厳密に平衡 → 検索が高速、挿入・削除はやや遅い

### 赤黒木

- 各ノードに「赤」か「黒」の色を付与し、以下の規則を維持
  1. 根は黒
  2. 赤ノードの子は必ず黒
  3. 根から葉までの黒ノード数は全経路で同じ（黒高さ）
- 高さ保証：h ≤ 2 log₂(n+1)
- AVL木より平衡が緩い → 挿入・削除の回転が少なく実装コスト低め
- **実用例**：Linux カーネルのスケジューラ、Java `TreeMap`、C++ `std::map`

## 計算量・パフォーマンス特性

| 操作 | 平均 (BST) | 最悪 (BST) | AVL木 | 赤黒木 |
|------|-----------|-----------|-------|--------|
| 検索 | O(log n) | O(n) | O(log n) | O(log n) |
| 挿入 | O(log n) | O(n) | O(log n) | O(log n) |
| 削除 | O(log n) | O(n) | O(log n) | O(log n) |
| 空間 | O(n) | O(n) | O(n) | O(n) |

- AVL木：回転回数 挿入O(1)回、削除O(log n)回
- 赤黒木：回転回数 挿入・削除ともにO(1)回（償却）→ 挿入・削除が多い場合に有利

## コード例

```python
class BSTNode:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        def _insert(node, val):
            if not node:
                return BSTNode(val)
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

Python標準ライブラリに赤黒木は公開されていないが、`sortedcontainers.SortedList` が内部的に類似の平衡構造を提供する。

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**：ソート済みデータを挿入すると高さがO(n)になる
- **AVL木が最良とは限らない**：書き込みが多いユースケースでは赤黒木の方が高速
- **削除は実装が複雑**：BSTの削除は「後継者ノード（右部分木の最小値）」への置き換えが必要
- **Pythonの`dict`や`set`はBSTではない**：ハッシュテーブル実装。順序を保証しない
- **重複値の扱い**：ツリーの設計次第（左 or 右、無視するなど）で挙動が変わる

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon (PostgreSQL) のインデックス**：デフォルトのB-Treeインデックスは赤黒木の考え方を発展させた多分木。`CREATE INDEX` の仕組みを理解する土台になる
- **範囲クエリの効率**：`WHERE created_at BETWEEN '2026-01-01' AND '2026-06-30'` はBSTの順序性があって初めてO(log n)のスキャンが可能
- **FastAPIのルーティング**：内部でトライ木（Radix木）を使ったルート解決。木構造の考え方が共通
- **ソート済み結果が必要な場合**：`SortedList` など平衡木ベースの構造は、ヒープより順序アクセスが得意（Kth要素取得、範囲検索）

### 使い分けの指針

| ユースケース | 推奨 |
|-------------|------|
| DB インデックス | B-Tree（BSTの多分木拡張） |
| 順序付き集合・辞書 | 赤黒木（言語標準ライブラリ） |
| 検索重視、更新少 | AVL木 |
| 優先度付き取り出し | ヒープ（BSTではなく次回テーマ） |
