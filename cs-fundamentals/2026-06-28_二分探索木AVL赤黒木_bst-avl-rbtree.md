# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

順序付きデータの高速な検索・挿入・削除を実現する木構造。配列の二分探索は検索O(log n)だが挿入がO(n)、連結リストは挿入O(1)だが検索がO(n)。木構造はその中間を取り、全操作をO(log n)に近づける。RDBMSのインデックス、標準ライブラリのSortedSet/TreeMapに内部採用されている。

---

## 仕組みの要点

### 二分探索木（BST: Binary Search Tree）

- **性質**: 左の子 < ノード < 右の子
- **検索**: ルートから比較しながら左右に下る → O(h)（hは木の高さ）
- **挿入**: 検索で末端に到達し葉として追加
- **問題点**: 挿入順次第で木が偏る（ソート済みデータを挿入→O(n)の線形リスト化）

```
挿入順 1,2,3,4 の場合（最悪ケース）:
1
 \
  2
   \
    3
     \
      4  ← 線形リストと同じ
```

### AVL木（高さ平衡二分木）

- **性質**: 任意ノードの左右部分木の高さの差が最大1
- **平衡係数**: `balance = height(左) - height(右)` → -1, 0, 1 の範囲を維持
- **回転操作**: 不均衡になったら左回転・右回転・左右回転・右左回転で修復
- **保証**: 常に高さ O(log n)、全操作O(log n)
- **弱点**: 挿入・削除のたびに回転が多発 → 更新が多いユースケースでは赤黒木より遅い

```
右回転の例（左が重い時）:
    z              y
   /              / \
  y      →       x   z
 /
x
```

### 赤黒木（Red-Black Tree）

- **性質**: 各ノードに赤/黒の色属性を付加し、以下の制約を維持
  1. ルートは黒
  2. 赤ノードの子は必ず黒（赤が連続しない）
  3. 任意ノードから葉までの黒ノード数は等しい（黒高さ一定）
- **高さ**: 最悪でも `2 * log(n+1)` → O(log n)
- **回転**: AVL木より回転・再配色の回数が少ない
- **採用例**: Linux カーネル（プロセス管理）、C++ `std::map`、Java `TreeMap`

---

## 計算量まとめ

| 操作 | BST（平均） | BST（最悪） | AVL木 | 赤黒木 |
|------|------------|------------|-------|--------|
| 検索 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除 | O(log n)   | O(n)       | O(log n) | O(log n) |
| 空間 | O(n)       | O(n)       | O(n)  | O(n)   |

- **AVL vs 赤黒木**: AVL木は厳密に平衡 → 検索が微妙に速い。赤黒木は更新コストが低い → 挿入・削除が多い場面で有利

---

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
        def _ins(node, v):
            if not node:
                return Node(v)
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

Pythonの`sortedcontainers.SortedList`は内部的に平衡木ライクな実装でO(log n)を保証。

---

## よくある誤解・落とし穴

- **「BSTは常にO(log n)」は誤り**: ランダムデータなら平均O(log n)だが、ソート済み入力でO(n)に劣化
- **AVL木は万能ではない**: 挿入・削除が頻繁なら赤黒木の方が実装コスト・実行コスト共に低い
- **削除は挿入より複雑**: 削除したノードを「中順後継ノード（右部分木の最小値）」で置換する手順が必要
- **Pythonの`dict`はBSTではない**: ハッシュテーブル実装。順序保証・範囲クエリが必要なら`SortedList`を使う

---

## 実務での使いどころ（FastAPI + Neon + Cloud Runスタック）

- **Neon (PostgreSQL) のインデックス**: `CREATE INDEX`はB-Tree（BSTの発展形）。範囲クエリ（BETWEEN、ORDER BY）に強い。等値検索のみならハッシュインデックスの方が速い場合もある
- **FastAPI でのソート済みデータ管理**: メモリ上でソート順を維持したいなら`sortedcontainers.SortedList`を活用。再挿入のたびにソートし直すより効率的
- **Cloud Run のオートスケール**: スケジューラ内部でプロセス優先度管理にヒープ・赤黒木を使用。OS層の知識としてトラブルシュートに役立つ
- **APIレスポンスのランキング**: Top-K取得はソート(O(n log n))より最大ヒープ(O(n log k))が速い。木構造の知識が最適化の判断に直結する
