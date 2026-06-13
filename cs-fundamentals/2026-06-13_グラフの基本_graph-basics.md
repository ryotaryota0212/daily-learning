# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で関係を表すデータ構造。SNSのフォロー関係、経路探索、依存関係の解析など、現実の問題の多くはグラフに帰着できる。BFS/DFSは全グラフアルゴリズムの土台であり、Dijkstra・トポロジカルソート・最小全域木など応用が広い。

## 仕組みの要点

### グラフの種類

- **有向グラフ（Directed）**: エッジに向きあり（例: フォロー関係）
- **無向グラフ（Undirected）**: 向きなし（例: 友達関係）
- **重み付きグラフ**: エッジに重み（コスト・距離）あり
- **DAG（有向非巡回グラフ）**: サイクルなしの有向グラフ → 依存関係の表現に使う

### グラフの表現方法

**隣接リスト（Adjacency List）**

```
graph = {
  0: [1, 2],
  1: [0, 3],
  2: [0, 3],
  3: [1, 2]
}
```

- 空間: O(V + E)
- エッジ有無の確認: O(degree) ← 隣接ノード数分スキャン
- 疎グラフに適する（実務でほぼこれ）

**隣接行列（Adjacency Matrix）**

```
matrix[i][j] = 1  # i→j のエッジあり
```

- 空間: O(V²)
- エッジ有無の確認: O(1)
- 密グラフ・重み付きグラフに有効。V が大きいとメモリを大量消費

### BFS（幅優先探索）

- キューを使って近い順に探索
- **最短ステップ数**（重みなし）を求めるときに使う
- 訪問済みを`visited`で管理してサイクルを防ぐ

```python
from collections import deque

def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```

### DFS（深さ優先探索）

- スタック（または再帰）で深く潜ってから戻る
- **連結判定・サイクル検出・トポロジカルソート**に使う

```python
def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
    return visited
```

## 計算量・パフォーマンス特性

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| 空間 | O(V+E) | O(V²) |
| BFS/DFS | O(V+E) | O(V²) |
| エッジ検索 | O(degree) | O(1) |
| 全隣接ノード列挙 | O(degree) | O(V) |

- BFS/DFS はどちらも O(V+E)。隣接行列ベースだと O(V²) になる
- 再帰DFSはスタック深さが O(V) → 大きいグラフで再帰制限に注意

## よくある誤解・落とし穴

- **`visited` の管理漏れ**: サイクルがあると無限ループ。必ずキューに入れる前（BFS）またはノード処理前（DFS）にマークする
- **BFSが最短路を保証するのは重みなしのとき**: 重み付きは Dijkstra を使う
- **再帰DFSの深さ制限**: Pythonのデフォルトは1000。大グラフでは`sys.setrecursionlimit`か反復DFSに切り替える
- **隣接行列の選択ミス**: V=10万ノードで隣接行列を作ると100億セル → メモリ爆発

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**

- **依存関係の解析**: マイクロサービス間の呼び出し関係をDAGで表し、循環依存の検出にDFS（サイクル検出）を使う
- **Neonのスキーマ設計**: 多対多リレーションをER図で整理するとき、グラフとして考えるとクエリ設計がしやすい
- **Cloud Run のデプロイ順序**: サービス間依存をトポロジカルソートで順序付けして安全にデプロイ
- **レコメンド・検索**: ユーザー行動グラフをBFSで近傍ノード探索（類似ユーザーや関連コンテンツ）
- **APIのレート制限チェック**: リクエストパスをグラフで追跡し、ボトルネックノードを特定
