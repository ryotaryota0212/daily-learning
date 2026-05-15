# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）は「左の子 < 親 < 右の子」という順序制約を持つ木構造。
探索・挿入・削除がO(log n)で行えるが、偏りが生じるとO(n)に劣化する。
AVL木・赤黒木はこの劣化を防ぐ「自己平衡木」で、データベースのインデックス（B-Tree）や言語標準ライブラリの内部実装に使われる。
「なぜDBインデックスで範囲検索が速いのか」「Pythonの`sortedcontainers`が何をしているか」を理解する鍵。

---

## 仕組みの要点

### 二分探索木（BST）

```
      5
     / \
    3   7
   / \   \
  1   4   9
```

- **探索**: ルートから比較しながら左右に降りる
- **挿入**: 探索と同じ手順で末端に追加
- **削除**: 子が2つある場合は「中順後継者（右部分木の最小値）」と入れ替えてから削除
- **弱点**: 昇順データを挿入すると右に偏り、木の高さがO(n)になる

```
1 → 2 → 3 → 4 → 5  # 連結リストと同じ構造になってしまう
```

---

### AVL木（高さ平衡木）

- **平衡条件**: 全ノードで `|左の高さ - 右の高さ| ≤ 1`
- **回転操作**: 挿入・削除後に条件が崩れたら回転で修正
  - 右回転（LL回転）: 左に偏ったとき
  - 左回転（RR回転）: 右に偏ったとき
  - 左右回転（LR回転）・右左回転（RL回転）: 2段階で修正
- **高さ保証**: 常にO(log n)
- **特徴**: 厳密な平衡 → 探索は速いが挿入・削除の回転コストがやや高い

```
挿入前:      挿入(6)後:      右回転後:
    5            5               5
   / \          / \             / \
  3   7        3   7           3   6
              /   / \             / \
             2   6   9           5   7
                                      \
                                       9
```

---

### 赤黒木（Red-Black Tree）

- **着色条件**（5つのルール）:
  1. 各ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの黒ノード数は同じ（黒高さ一定）
- **保証**: 高さは最大 `2 * log(n+1)` → O(log n)
- **AVLとの違い**: 厳密な平衡より「ゆるい平衡」。回転回数が少なく挿入・削除が速い
- **実用**: C++ `std::map`、Java `TreeMap`、Linuxカーネルのプロセス管理に採用

---

## 計算量・パフォーマンス特性

| 操作     | BST（平均） | BST（最悪） | AVL木  | 赤黒木 |
|----------|------------|------------|--------|--------|
| 探索     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 挿入     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 削除     | O(log n)   | O(n)       | O(log n) | O(log n) |
| 回転回数 | -          | -          | O(log n) | O(1)（平均） |

- **AVL**: 探索頻度が高いとき有利（厳密な平衡で木が低い）
- **赤黒木**: 挿入・削除が多いとき有利（回転が少ない）

---

## コード例（Python: BST の探索・挿入）

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

- **「BSTは常にO(log n)」は誤り**: ランダムデータなら平均O(log n)だが、ソート済みデータでO(n)になる
- **「AVL木は赤黒木より優れている」は誤り**: 用途次第。探索重視ならAVL、書き込み重視なら赤黒木
- **削除の複雑さを見落とす**: 子が2つあるノードの削除は「中順後継者の探索→置換→削除」と3ステップ
- **B-TreeとBSTを混同**: DBのインデックスはB-Tree（多分岐、ディスクI/O最適化）であり、赤黒木とは別物
- **平衡木でも最悪O(log n)は保証だがO(1)ではない**: キャッシュ効率はハッシュテーブルに劣る

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neon（PostgreSQL）のインデックス**: `CREATE INDEX`で作られるB-Treeは平衡木の応用。  
  「なぜ`WHERE age > 30 ORDER BY age`が速いか」はB-Treeが順序を保持しているから
  ```sql
  -- 範囲検索でインデックスが効く（B-Treeの中順走査を利用）
  EXPLAIN SELECT * FROM users WHERE age BETWEEN 20 AND 40;
  ```
- **Python `sortedcontainers.SortedList`**: 内部はB-Tree的な構造。  
  FastAPIで「上位N件をリアルタイム集計したい」場面に有効
- **Cloud Runのリクエストルーティング**: 内部のロードバランサーはハッシュや平衡木でルート管理
- **APIのレート制限実装**: 「ユーザーIDでソートされた挿入・削除」に赤黒木ベースの`SortedDict`が適合

### 覚えておくべき判断軸

| シナリオ | 選択 |
|---------|------|
| キーの順序・範囲検索が必要 | 平衡木（or B-Tree） |
| 検索のみで挿入削除が少ない | AVL木 |
| 挿入削除が多い | 赤黒木 |
| 順序不要・点探索のみ | ハッシュテーブル |
