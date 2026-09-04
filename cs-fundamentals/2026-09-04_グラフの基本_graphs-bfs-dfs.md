# グラフの基本（表現方法、BFS、DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で構成されるデータ構造。SNSの友人関係、経路探索、依存関係解決など実務頻出の問題の多くがグラフとして表現できる。BFS・DFSは「グラフをどう探索するか」の基本戦略で、最短経路・連結判定・トポロジカルソートの土台となる。

---

## 仕組みの要点

### グラフの種類
- **有向グラフ**: エッジに向きあり。依存関係、フォロー関係など
- **無向グラフ**: エッジは双方向。友人関係、道路ネットワークなど
- **DAG（有向非巡回グラフ）**: 閉路なし。タスク依存・ビルドシステムに多用

### 表現方法
- **隣接リスト** `dict[node] = [neighbors]`：空間O(V+E)。疎グラフ向き（実務の主流）
- **隣接行列** `matrix[u][v]`：空間O(V²)。辺確認O(1)だがV=10万で10GBになる

### BFS（幅優先探索）
- **キュー**を使い、近いノードから順に探索
- 重みなしグラフで**最短ホップ数**を保証
- 探索順: 起点 → 距離1の全ノード → 距離2の全ノード → ...

### DFS（深さ優先探索）
- **スタック（または再帰）** を使い、一方向に深く進む
- **連結成分の検出・サイクル検出・トポロジカルソート**に向く
- 深いグラフでは再帰上限（Pythonはデフォルト1000）に注意

---

## 計算量

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| BFS / DFS | O(V+E) | O(V²) |
| 辺の確認 | O(degree) | O(1) |

---

## コード例（Python）

```python
from collections import deque, defaultdict

graph = defaultdict(list)
for u, v in [(1,2),(1,3),(2,4),(3,4),(4,5)]:
    graph[u].append(v); graph[v].append(u)

def bfs(start):
    dist = {start: 0}
    q = deque([start])
    while q:
        node = q.popleft()
        for nb in graph[node]:
            if nb not in dist:
                dist[nb] = dist[node] + 1
                q.append(nb)
    return dist  # {1:0, 2:1, 3:1, 4:2, 5:3}

def dfs(node, visited=None):
    visited = visited or set()
    visited.add(node)
    for nb in graph[node]:
        if nb not in visited:
            dfs(nb, visited)
    return visited  # {1,2,3,4,5}
```

---

## よくある誤解・落とし穴

- **BFSは常に最短経路**: 重みなしグラフのみ。重み付きはDijkstraが必要
- **DFSは再帰で十分**: 深いグラフで再帰上限に当たる。明示的スタックが安全
- **visited管理を忘れる**: 無向グラフで無限ループに陥る。必ずvisited集合を使う

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連:**

- **依存関係解決**: マイクロサービスの起動順序をDAG+トポロジカルソートで管理
- **ユーザー推薦**: Neon上のフォローテーブルで2-hopグラフ探索→「知り合いかも」機能
- **障害伝播分析**: Cloud Runサービス間の呼び出しグラフで影響範囲を特定
- **外部キー循環チェック**: テーブルの循環参照検出はDFSのサイクル検出と同じ問題
