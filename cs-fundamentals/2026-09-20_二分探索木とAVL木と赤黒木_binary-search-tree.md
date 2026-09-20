# 二分探索木・AVL木・赤黒木の基本と計算量

## 概要

二分探索木（BST）はキーの大小関係を維持する木構造で、検索・挿入・削除を効率的に行える。  
しかし素朴なBSTは入力順次第で最悪O(n)に劣化する。これを防ぐためAVL木・赤黒木などの**自己平衡BST**が生まれた。  
データベースのインデックス（B-Tree）やOS内部の多くのデータ構造に応用されており、「なぜインデックスがあると速いか」の根拠理解に直結する。

---

## 仕組みの要点

### 二分探索木（BST）の基本則

- 各ノードの左サブツリーには「より小さい値」、右には「より大きい値」
- 中順巡回（in-order traversal）すると昇順でキーが取得できる
- 挿入順が偏る（例: 昇順データ）と木が一方向に伸びてリスト化する → 最悪O(n)

```
    5
   / \
  3   7
 / \
1   4
```

### AVL木

- 各ノードで「左右サブツリーの高さ差（バランス因子）が ±1 以内」を維持
- 違反した場合、**回転（rotation）** で再平衡化（4パターン: LL/RR/LR/RL）
- 挿入・削除のたびに回転が必要なため書き込みコストがやや高い
- 読み取り多め・書き込み少なめのユースケースに向く

```
# 右回転（LL回転）のイメージ
    C (BF=2)          B
   /          →      / \
  B                 A   C
 /
A
```

### 赤黒木

- 各ノードを「赤 or 黒」に色付けし、以下の5条件を維持:
  1. 全ノードは赤か黒
  2. ルートは黒
  3. 葉（NIL）は黒
  4. 赤ノードの子は必ず黒（赤が連続しない）
  5. 任意のノードから葉までの黒ノード数は同一（黒高さ一定）
- AVL木より高さがやや高くなりうるが、回転回数が最大3回に抑えられ書き込みが速い
- Java の `TreeMap`、Linux カーネルのスケジューラ（CFS）、Nginx タイマーなどに採用

---

## 計算量・パフォーマンス特性

| 操作        | 素朴なBST | AVL木     | 赤黒木    |
|-----------|---------|---------|---------|
| 検索        | O(h)    | O(log n) | O(log n) |
| 挿入        | O(h)    | O(log n) | O(log n) |
| 削除        | O(h)    | O(log n) | O(log n) |
| 高さ上限      | O(n)    | 1.44 log n | 2 log n |
| 挿入時の回転上限 | 0       | O(log n) | 2回      |
| 削除時の回転上限 | 0       | O(log n) | 3回      |

- h = 木の高さ（最悪ケースでO(n)になりうる）
- 空間計算量: いずれも O(n)

---

## コード例（Python）

```python
class BSTNode:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None

class BST:
    def insert(self, root, key):
        if root is None:
            return BSTNode(key)
        if key < root.key:
            root.left = self.insert(root.left, key)
        elif key > root.key:
            root.right = self.insert(root.right, key)
        return root

    def search(self, root, key):
        if root is None or root.key == key:
            return root
        if key < root.key:
            return self.search(root.left, key)
        return self.search(root.right, key)
```

実用では `sortedcontainers.SortedList`（Python）や `std::map`（C++）が内部で平衡BSTを使う。

---

## よくある誤解・落とし穴

- **「BSTはO(log n)で動く」は間違い**: 入力が偏ると最悪O(n)。保証はAVL/赤黒木を使うとき
- **AVL木の方が常に速い**: 検索だけなら高さが低いAVL木が有利だが、挿入・削除の多いケースでは赤黒木の方がスループットが出ることが多い
- **赤黒木の条件を暗記しようとする**: 本質は「最長パスが最短パスの2倍以内になる」という高さ保証にある
- **削除が検索より複雑**: 削除は後継ノード（右サブツリーの最小値）への置換が必要で、実装バグが出やすい
- **B-Treeと混同**: データベースが使うB-TreeはBSTとは別物（nノードの子を持つ多分木）

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

**Neon（PostgreSQL）のインデックス内部**  
- `CREATE INDEX` はデフォルトでB-Tree（BSTの多分岐版）を作る
- `ORDER BY`・範囲検索（`BETWEEN`、`>`、`<`）が速いのはB-Treeが順序を保持するから
- ハッシュインデックスと違い、**範囲検索・ソート済み結果**に強い理由が平衡木の性質で理解できる

```sql
-- B-Treeインデックスが効くクエリ（範囲 + ソート）
SELECT * FROM posts WHERE created_at > '2026-01-01' ORDER BY created_at;

-- ハッシュインデックスが効くクエリ（等値のみ）
SELECT * FROM users WHERE email = 'foo@example.com';
```

**FastAPI でのユースケース**  
- インメモリキャッシュでキー順アクセスが必要なとき `SortedList` / `SortedDict` を検討
- リーダーボード・タイムライン機能: Neonの範囲クエリ + インデックスで十分なことが多く、自前BST実装は不要
- ただし「なぜインデックスが効くか」を理解しているとクエリ設計・EXPLAINの読み方が体感で分かる
