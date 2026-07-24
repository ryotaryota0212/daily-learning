# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）系のデータ構造は、「順序を保ちながら高速に検索・挿入・削除」が必要な場面で使われる。
Pythonの`sortedcontainers`、JavaのTreeMap、PostgreSQLのB-Treeインデックスの根幹にある考え方。
ただし素朴なBSTは最悪O(n)に退化するため、自己平衡木（AVL・赤黒木）が実務では使われる。

---

## 仕組みの要点

### 二分探索木（BST）の基本

- 各ノードは「左の子 < 自分 < 右の子」を常に満たす
- 検索: ルートから値を比較し、左右に降りていく → 平均 O(log n)
- **問題**: 昇順でデータを挿入すると木が一直線（連結リスト化）→ 最悪 O(n)

```
挿入順: 1, 2, 3, 4, 5
1
 \
  2
   \
    3  ← 探索がO(n)に退化
     \
      4
```

### AVL木（高さ平衡）

- 各ノードで「左右の部分木の高さ差 ≤ 1」を保証（平衡因子）
- 挿入・削除後にバランスが崩れたら**回転**操作で修正
  - 右回転 / 左回転 / 二重回転（LR回転・RL回転）
- 高さを厳密に管理 → 検索が赤黒木より少し速い
- 挿入・削除のたびに回転が多い → 書き込みが多い場面ではやや重い

### 赤黒木（色による緩い平衡）

- 各ノードに「赤」か「黒」の色を付与し、以下の4条件を維持
  1. ルートは黒
  2. 赤ノードの子は黒（赤が連続しない）
  3. 全葉（NIL）から根までの黒ノード数が同じ
  4. 全ノードは赤か黒
- AVL木より平衡条件が緩い → 挿入・削除時の回転が少ない
- 高さはO(2 log n)まで許容 → 検索はAVLより若干遅いが実用上は同等
- **Linuxカーネル、Java TreeMap、C++ std::map** が採用

---

## 計算量

| 操作     | BST（平均） | BST（最悪） | AVL木    | 赤黒木   |
|---------|-----------|-----------|---------|---------|
| 検索    | O(log n)  | O(n)      | O(log n)| O(log n)|
| 挿入    | O(log n)  | O(n)      | O(log n)| O(log n)|
| 削除    | O(log n)  | O(n)      | O(log n)| O(log n)|
| 空間    | O(n)      | O(n)      | O(n)    | O(n)    |

- AVL木: 高さ ≤ 1.44 log(n+2)、赤黒木: 高さ ≤ 2 log(n+1)

---

## コード例（Python: シンプルなBST）

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

class BST:
    def __init__(self): self.root = None

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

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ランダムデータなら平均O(log n)だが、ソート済みデータで最悪O(n)
- **AVL木の方が赤黒木より常に速い**: 検索は若干速いが、挿入・削除は赤黒木の方が速い場合が多い
- **平衡木とB-Treeは別物**: データベースのインデックスはB-Tree（ディスクI/O最適化のため複数キーを1ノードに格納）
- **Python標準ライブラリにBSTはない**: `sortedcontainers.SortedList`（Cで実装済み）を使うのが実務的

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**

- **Neonのインデックス**: `CREATE INDEX`で作るB-Treeインデックスは赤黒木の考えを発展させた構造。EXPLAINでIndex Scanが選ばれる条件（選択率 < 10〜20%）を理解するのに必要
- **範囲クエリの最適化**: BSTは「中順探索 = ソート済みイテレーション」が得意 → `BETWEEN`や`ORDER BY`がインデックスを活かせる理由
- **APIレイテンシのソート・フィルタ**: FastAPI側でソート済みコレクションが必要な場合、Pythonの`sortedcontainers`で挿入しながら順序維持（全件ソートより効率的）
- **Cloud Runのコールドスタート**: 起動時に大量データをメモリに読み込むなら、リスト全ソートより挿入しながら平衡木に積む方がスループット改善の可能性あり
