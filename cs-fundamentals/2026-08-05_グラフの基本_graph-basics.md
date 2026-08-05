# グラフの基本（表現方法、BFS、DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で関係を表すデータ構造。SNSの友人関係、依存関係の解決、経路探索など実務に広く登場する。「BFSは最短距離、DFSは全探索・閉路検出」という使い分けを理解することで、未知の問題をグラフに落とし込んで解けるようになる。

## 仕組みの要点

### グラフの種類
- **有向グラフ（Directed）**: エッジに向きあり。依存関係、フォロー関係
- **無向グラフ（Undirected）**: エッジに向きなし。友人関係、道路ネットワーク
- **重み付きグラフ（Weighted）**: エッジにコストあり。経路コスト最適化

### 表現方法の比較

| 方法 | 空間 | 隣接ノード取得 | 辺の存在確認 | 向いているケース |
|------|------|--------------|------------|----------------|
| 隣接リスト | O(V+E) | O(degree) | O(degree) | 疎なグラフ（推奨） |
| 隣接行列 | O(V²) | O(V) | O(1) | 密なグラフ |

- 実務では**隣接リスト**が基本。Pythonでは`defaultdict(list)`が便利

### BFS（幅優先探索）の動作
1. スタートノードをキューに入れ、visitedに追加
2. キューからノードを取り出し、未訪問の隣接ノードをキューへ
3. 繰り返すことで、スタートから**層ごと**に探索
- 最初にゴールに到達したとき = **最短距離**（エッジの重みが均一なとき）

### DFS（深さ優先探索）の動作
1. スタートノードから1つの隣接ノードへ潜る
2. 行き止まりになったら戻る（バックトラック）
3. 再帰またはスタックで実装
- 全経路列挙、閉路検出、トポロジカルソートに使う

## 計算量・パフォーマンス特性

| 操作 | 時間計算量 | 備考 |
|------|-----------|------|
| BFS | O(V + E) | V=頂点数、E=辺数 |
| DFS | O(V + E) | 再帰の場合スタック深さに注意 |
| 隣接リストの構築 | O(E) | |

- 再帰DFSは深いグラフ（V > 1000）でスタックオーバーフローのリスク → 反復に切り替え

## コード例（Python）

```python
from collections import defaultdict, deque

graph = defaultdict(list)
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

- **BFS = 必ず最短距離ではない**: エッジに重みがある場合はDijkstra法が必要。重み均一なときのみBFSで最短距離
- **visitedチェックを後回しにするとTLE**: キューに入れるときにvisitedに追加しないと同じノードを複数回キューに積む
- **有向グラフと無向グラフを混同**: 「A→B」の辺があっても「B→A」は存在しない（有向）
- **再帰DFSの深さ制限**: Pythonのデフォルト再帰制限は1000。`sys.setrecursionlimit`か反復実装で対処

## 実務での使いどころ

### FastAPI + Neon + Cloud Runスタックとの関連

- **依存関係の解決**: マイクロサービス間やモジュール間の依存をDAG（有向非巡回グラフ）で表現。起動順序の決定にトポロジカルソート（DFS応用）
- **クエリの結合経路探索**: Neonでテーブル間のJOINを設計するとき、テーブルをノード・外部キーをエッジとして関係を可視化
- **Cloud Runのルーティング**: API間の呼び出し連鎖をグラフとして捉え、循環依存（閉路）を早期に発見
- **レコメンデーション**: ユーザー・コンテンツの関係グラフをBFSで2ホップ以内のノードを取得してレコメンド候補に
