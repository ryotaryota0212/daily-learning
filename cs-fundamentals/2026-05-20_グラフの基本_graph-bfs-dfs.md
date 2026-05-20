# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフは「頂点（ノード）」と「辺（エッジ）」で関係性を表すデータ構造。
SNSのフォロー関係・経路探索・依存関係解析など、現実の問題はグラフとして定式化できることが多い。
BFS/DFSは最も基本的な探索手法であり、ダイクストラ法・トポロジカルソート・連結成分検出など多くのアルゴリズムの土台になる。
FastAPIのルーティング依存関係や、Neonのテーブル参照関係もグラフ構造として捉えられる。

---

## 仕組みの要点

### グラフの種類
- **有向グラフ（Directed）**: 辺に向きがある。Twitterのフォロー、タスク依存関係
- **無向グラフ（Undirected）**: 辺に向きなし。友人関係、道路ネットワーク
- **重み付きグラフ**: 辺にコスト。距離・料金・レイテンシなど

### 表現方法の比較

| 表現 | 空間計算量 | 隣接確認 | 隣接リスト取得 | 向いているケース |
|------|-----------|---------|--------------|----------------|
| 隣接行列 | O(V²) | O(1) | O(V) | 密なグラフ、頂点数が少ない |
| 隣接リスト | O(V+E) | O(degree) | O(degree) | 疎なグラフ（実務の多数） |

```
隣接リスト例（無向グラフ）:
  0 - 1, 2
  1 - 0, 3
  2 - 0
  3 - 1

graph = {0: [1,2], 1: [0,3], 2: [0], 3: [1]}
```

### BFS（幅優先探索）
- キューを使い、始点から近い順に探索
- **最短経路（辺の重みが均一な場合）** を保証
- 処理順: 層（level）ごとに広がる

### DFS（深さ優先探索）
- スタック（or 再帰）を使い、できるだけ深く進む
- **連結判定・閉路検出・トポロジカルソート** に向く
- 処理順: 一本道を最深まで掘ってからバックトラック

---

## 計算量

| 操作 | 計算量（隣接リスト） |
|------|-------------------|
| BFS / DFS | O(V + E) |
| 隣接行列でのBFS/DFS | O(V²) |
| 空間（訪問済み管理） | O(V) |

V = 頂点数、E = 辺数

---

## コード例（Python）

```python
from collections import deque

graph = {0: [1,2], 1: [0,3], 2: [0], 3: [1]}

def bfs(start):
    visited = set([start])
    queue = deque([start])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order

def dfs(node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    result = [node]
    for neighbor in graph[node]:
        if neighbor not in visited:
            result += dfs(neighbor, visited)
    return result

print(bfs(0))  # [0, 1, 2, 3]
print(dfs(0))  # [0, 1, 3, 2]
```

---

## よくある誤解・落とし穴

- **訪問済み管理を忘れると無限ループ**（特に閉路がある有向グラフ）
- BFSの「最短経路」保証は**辺の重みがすべて等しい場合のみ**。重みが異なる場合はダイクストラ法が必要
- DFSの再帰実装は**スタックオーバーフローに注意**（深さが数万を超えるケース）。itertools的な反復実装に切り替える
- 有向グラフで「到達可能か」と「双方向到達可能か」は別問題。混同しやすい

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **依存関係の解決**: FastAPIの`Depends()`チェーンはDAG（有向非巡回グラフ）。`DFS`でトポロジカルソートして初期化順を決定
- **SQLのJOINパス探索**: テーブル間のFKをグラフとして見ると、クエリのJOINパスがBFSで見つかる
- **Cloud Runのサービスメッシュ**: マイクロサービス間の呼び出し関係をグラフ化し、循環依存（閉路）を検出
- **API制限のレート計算**: リクエストフローをグラフで表現し、ボトルネックをBFSで特定
- **Neonのブランチ管理**: Neonのブランチ構造は木（グラフの特殊形）。DFSで全ブランチを走査して古いブランチを検出できる
