# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で構成されるデータ構造。
SNSの友人関係、経路探索、依存関係解決など、実務の多くの問題がグラフとして自然にモデル化できる。
BFS/DFSという2つの探索戦略を理解すると、「最短経路」「到達可能か判定」「トポロジカル順序」など幅広い問題が解ける。

## 仕組みの要点

### グラフの種類
- **有向グラフ（Directed）**: エッジに向きあり（フォロー関係、依存関係）
- **無向グラフ（Undirected）**: エッジに向きなし（友人関係、道路網）
- **重み付きグラフ**: エッジにコスト付き（地図の距離、通信遅延）
- **DAG（Directed Acyclic Graph）**: 循環なし有向グラフ（タスク依存、パッケージ依存）

### グラフの表現方法

**隣接リスト（Adjacency List）**
```
graph = {
  A: [B, C],
  B: [D],
  C: [D, E],
  D: [],
  E: []
}
```
- メモリ: O(V + E)
- エッジ確認: O(degree)
- **疎なグラフに適する**（実務のほとんどはこちら）

**隣接行列（Adjacency Matrix）**
```
  A B C D E
A 0 1 1 0 0
B 0 0 0 1 0
C 0 0 0 1 1
```
- メモリ: O(V²)
- エッジ確認: O(1)
- **密なグラフ・頻繁なエッジ参照に適する**

### BFS（幅優先探索）

- キューを使い、近い順に探索
- **最短経路（辺の重みが均一）を保証**
- 訪問済みセットで無限ループ防止

```python
from collections import deque

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
- **連結成分の検出、サイクル検出、トポロジカルソート**に活用

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
| BFS / DFS | O(V + E) | O(V²) |
| エッジ追加 | O(1) | O(1) |
| エッジ確認 | O(degree) | O(1) |
| メモリ | O(V + E) | O(V²) |

- V = 頂点数、E = 辺数
- 実務のグラフは疎（E << V²）が多いため、隣接リスト + BFS/DFS が O(V + E) で効率的

## よくある誤解・落とし穴

- **訪問済み管理を忘れる**: サイクルがあると無限ループ。`visited` セットは必須
- **BFSが最短路を保証するのは重みなし限定**: 重みあり最短路はダイクストラ法が必要
- **再帰DFSはスタックオーバーフロー**: 深いグラフでは反復版DFS（明示的スタック）を使う
- **有向グラフの到達可能性は非対称**: A→B でも B→A とは限らない

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連:**

- **パッケージ依存解決**: `pip` や npm がパッケージ依存をDAGとして管理し、トポロジカルソートでインストール順を決定
- **APIの認可チェック**: ロール階層（admin > editor > viewer）を有向グラフで表現し、DFSで権限継承を解決
- **Neonのクエリプラン**: PostgreSQLのクエリ最適化器がジョイン順序をグラフ探索で決定
- **Cloud Runのヘルスチェック**: サービス間依存関係のチェックにBFSを応用（どのサービスが到達不能か）
- **SNS機能の実装**: フォロワー推薦（友人の友人）はBFS 2ホップで実現可能
