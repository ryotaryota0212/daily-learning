# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）からなるデータ構造。SNSの友人関係・依存解決・ルート探索など、現実の問題の大半がグラフとして表現できる。BFS/DFSはグラフ探索の基本で、最短経路・連結判定・トポロジカルソートなどに応用が広い。

---

## 表現方法

### 隣接リスト（Adjacency List）
- ノードごとに「隣接ノードのリスト」を持つ
- 空間計算量: O(V + E)
- スパースグラフ（辺が少ない）に向く
- 実装: `graph = defaultdict(list)`

### 隣接行列（Adjacency Matrix）
- V×Vの2次元配列。`matrix[u][v] = 1` で辺を表現
- 空間計算量: O(V²)
- 辺の有無をO(1)で確認できるが、ノード数が大きいと非現実的
- デンスグラフ（辺が多い）に向く

**実務ではほぼ隣接リスト一択。**

---

## 探索アルゴリズム

### BFS（幅優先探索）
- キューを使い、近い順に探索
- **最短経路（辺の重みなし）** を保証する
- 計算量: O(V + E)

```python
from collections import deque, defaultdict

def bfs(graph, start):
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
```

### DFS（深さ優先探索）
- スタック（再帰）を使い、深く潜って探索
- サイクル検出・トポロジカルソート・連結成分数に使う
- 計算量: O(V + E)

```python
def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    for neighbor in graph[start]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
    return visited
```

---

## 計算量まとめ

| 操作 | 隣接リスト | 隣接行列 |
|---|---|---|
| 空間 | O(V+E) | O(V²) |
| 辺の存在確認 | O(degree) | O(1) |
| BFS/DFS | O(V+E) | O(V²) |
| 全隣接ノード列挙 | O(degree) | O(V) |

---

## よくある誤解・落とし穴

- **BFS = 最短経路は「重みなし」グラフのみ。** 重みありはDijkstraが必要
- **再帰DFSはスタックオーバーフロー** のリスク。ノード数が多いなら反復（明示的スタック）に切り替える
- **有向グラフの訪問済み管理を忘れると無限ループ**。`visited` セットは必須
- 無向グラフの辺を隣接リストに追加する際、`graph[u].append(v)` と `graph[v].append(u)` の両方を忘れずに

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

| シーン | 活用 |
|---|---|
| マイクロサービスの依存関係 | 有向グラフ + トポロジカルソートでデプロイ順を決定 |
| ルーティングテスト | URLパスをグラフとして全経路カバレッジを検証 |
| Neon（PostgreSQL）クエリ | テーブルの外部キー関係をグラフと見てJOINの順序を最適化 |
| Cloud Run サービス間通信 | サービスの依存グラフで循環依存を検出（DFS） |

最短経路やコネクション最適化はバックエンドでよく出現する。Dijkstraへの拡張もBFSの理解が前提。
