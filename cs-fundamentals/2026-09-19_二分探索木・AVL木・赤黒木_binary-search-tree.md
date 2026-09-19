# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左 < 親 < 右」の順序制約を持つ木構造で、検索・挿入・削除を効率的に行う基盤となる。しかし単純なBSTは偏りが生じると最悪 O(n) に劣化するため、自動的に木を再バランスする **AVL木** や **赤黒木** が実用上使われる。データベースのインデックス（B-Tree）やプログラミング言語の標準ライブラリ（Java の `TreeMap`、C++ の `std::map`）の多くは赤黒木ベース。「なぜハッシュテーブルではなく木を使うか」の答えは **順序保証** にある。

---

## 仕組みの要点

### 二分探索木（BST）の基本操作

```
      8
     / \
    3   10
   / \    \
  1   6   14
     / \
    4   7
```

- **検索**: 値を比較して左右どちらかへ降りる → 平均 O(log n)
- **挿入**: 空きリーフに追加 → 平均 O(log n)
- **削除**: 子が2つの場合は「中順後継者（右部分木の最小値）」と交換してから削除
- **問題点**: 昇順でデータを挿入すると一本鎖になり O(n) に退化

### AVL木

- **制約**: 任意のノードで「左右の高さの差（バランス因子）≤ 1」を強制
- **修正操作**: 挿入・削除後にバランス因子が ±2 になったら **回転** で修正
  - 右回転（Left-Left case）
  - 左回転（Right-Right case）
  - 左右二重回転（Left-Right case）
  - 右左二重回転（Right-Left case）
- **特徴**: 高さが厳密に制御されるため検索が速い。挿入・削除の回転コストはやや高め

### 赤黒木

- **制約（5つの性質）**:
  1. 各ノードは赤か黒
  2. 根は黒
  3. 葉（NILノード）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの「黒ノード数（黒高さ）」が等しい
- **高さの上限**: 黒高さを h とすると全体の高さ ≤ 2h → O(log n) を保証
- **AVLとの比較**: 制約がゆるい分、挿入・削除の回転回数が少ない（最大3回）

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均）| BST（最悪）| AVL木 | 赤黒木 |
|----------|-----------|-----------|-------|--------|
| 検索     | O(log n)  | O(n)      | O(log n) | O(log n) |
| 挿入     | O(log n)  | O(n)      | O(log n) | O(log n) |
| 削除     | O(log n)  | O(n)      | O(log n) | O(log n) |
| 最小/最大 | O(log n)  | O(n)      | O(log n) | O(log n) |

- **AVL木**: 高さ = 1.44 × log₂(n+2) → 赤黒木より背が低く検索に有利
- **赤黒木**: 高さ ≤ 2 × log₂(n+1) → 挿入・削除の回転数が少なく書き込みに有利
- **空間**: どちらも O(n)。赤黒木は各ノードに色ビット1bit追加

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

---

## よくある誤解・落とし穴

- **「BSTは常に O(log n)」**: ソート済みデータを挿入すると O(n) に退化する。実用では必ずAVL/赤黒木を使う
- **「赤黒木 = AVL木の上位互換」**: 用途次第。読み込み多なら検索性能が高いAVL木が優位
- **「削除が単純」**: 子が2つある削除は中順後継者の置き換えが必要。実装バグが起きやすい
- **「回転は木の値を変える」**: 回転は構造（リンク）を変えるだけで順序（BST性質）は保たれる
- **「B-TreeはBSTの変種」**: B-Treeはディスク向けで1ノードに複数キーを持つ別物

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX` はB-Tree（BSTの多分木版）がデフォルト。`ORDER BY` や範囲検索（`BETWEEN`、`>`、`<`）が高速なのはこの木構造のおかげ
  ```sql
  -- 範囲検索でB-Treeが効く例
  SELECT * FROM orders WHERE created_at BETWEEN '2026-01-01' AND '2026-09-19';
  ```
- **なぜハッシュインデックスでなく木か**: ハッシュは `=` のみ。`>` や `ORDER BY` には木が必要
- **FastAPI のレスポンスソート**: Python の `sorted()` は TimSort（O(n log n)）だが、挿入しながら順序を保ちたい場面（ランキング更新など）は `sortedcontainers.SortedList`（内部は B-Tree ライク）が有効
- **Cloud Run のメモリ効率**: 赤黒木はポインタ2本＋色1bit。数百万件のインメモリキャッシュには十分軽量
