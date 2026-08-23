# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で関係を表すデータ構造。SNSのフォロー関係、ルート探索、依存関係解決など実務の幅広い問題がグラフとして定式化できる。BFS・DFSは最短経路・到達可能性・サイクル検出など、多くのアルゴリズムの基盤になる。

## 仕組みの要点

### グラフの種類

- **有向グラフ**：エッジに方向あり（Aがフォロー→B）
- **無向グラフ**：エッジに方向なし（A-B間に道路）
- **重み付きグラフ**：エッジにコストあり（距離・時間）
- **DAG（有向非巡回グラフ）**：サイクルなし、トポロジカルソート可能

### 表現方法の比較

| 方法 | 空間計算量 | 辺の存在確認 | 隣接ノード列挙 | 向いている場面 |
|------|-----------|------------|--------------|--------------|
| 隣接行列 | O(V²) | O(1) | O(V) | 密グラフ、頻繁な辺確認 |
| 隣接リスト | O(V+E) | O(degree) | O(degree) | 疎グラフ（実務の大半） |

**隣接リスト**が実務では一般的。V=頂点数、E=辺数。

### BFS（幅優先探索）

- キューを使い、起点から近い順に探索
- **最短ホップ数**の探索に最適
- 処理順: 起点 → 距離1の全ノード → 距離2の全ノード…

### DFS（深さ優先探索）

- スタック（または再帰）を使い、深く進んでから戻る
- **到達可能性・サイクル検出・トポロジカルソート**に使う
- 状態管理: `unvisited → visiting → visited` の3状態で閉路を検出

## 計算量

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| BFS/DFS | O(V+E) | O(V²) |
| 辺の追加 | O(1) | O(1) |
| 辺の削除 | O(degree) | O(1) |

## コード例（Python）

```python
from collections import deque, defaultdict

graph = defaultdict(list)  # 隣接リスト（無向グラフ）
for u, v in [(0,1),(0,2),(1,3),(2,3),(3,4)]:
    graph[u].append(v)
    graph[v].append(u)

def bfs(start, goal):
    queue = deque([(start, [start])])
    visited = {start}
    while queue:
        node, path = queue.popleft()
        if node == goal:
            return path
        for nb in graph[node]:
            if nb not in visited:
                visited.add(nb)
                queue.append((nb, path + [nb]))
    return None

def dfs(node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    for nb in graph[node]:
        if nb not in visited:
            dfs(nb, visited)
    return visited

print(bfs(0, 4))   # [0, 1, 3, 4] or [0, 2, 3, 4]
print(dfs(0))      # {0, 1, 2, 3, 4}
```

## よくある誤解・落とし穴

- **visited管理の漏れ**：無向グラフで管理しないと無限ループ
- **BFSで最短経路は「ホップ数」のみ**：重み付きグラフにはダイクストラが必要
- **再帰DFSのスタックオーバーフロー**：深いグラフ（V>1000程度）では反復版に切り替える
- **隣接行列をデフォルトに選ぶ**：V=10万以上だとメモリがV²=100億バイト級になる

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連：**

- **依存関係解決**：Cloud Runのサービス起動順序をDAGとしてモデル化し、トポロジカルソートで決定
- **SQLクエリのJOINグラフ**：テーブル間の外部キー関係をグラフで把握するとクエリ設計がしやすい
- **APIエンドポイントの到達可能性チェック**：認証が必要なルートのグラフを構築し、DFSで保護漏れを検出
- **Neonのデータ探索**：ユーザー→リソース→タグのような多対多関係をグラフで設計するとBFS的なアクセスパターンが自然に実装できる
