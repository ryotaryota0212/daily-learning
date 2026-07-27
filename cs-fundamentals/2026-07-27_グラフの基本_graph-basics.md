# グラフの基本（表現方法・BFS・DFS）

## 概要
グラフはノード（頂点）とエッジ（辺）で「関係」を表すデータ構造。SNSの友人関係・経路探索・タスク依存関係など広く使われる。BFS・DFSを理解すると最短経路・到達可能性・ループ検出が実装できる。Webサービスでもサービス間依存やDB外部キー設計に直結する。

## 仕組みの要点

### グラフの種類
- **有向グラフ（Directed）**: A→B と B→A は別物（フォロー関係）
- **無向グラフ（Undirected）**: A-B は双方向（友達関係）
- **重み付きグラフ**: エッジにコスト付き（地図の距離、レイテンシ）
- **DAG（有向非巡回グラフ）**: ループなし → タスクスケジューリング、依存解決

### 表現方法の比較

| 方式 | メモリ | エッジ確認 | 向き |
|------|-------|-----------|------|
| 隣接リスト | O(V+E) | O(degree) | 疎グラフ向き |
| 隣接行列 | O(V²) | O(1) | 密グラフ向き |

- 実務では**隣接リスト**がほぼ標準（Webのグラフは疎）

### BFS（幅優先探索）
- **キュー**を使い、近い順（レベル順）に探索
- エッジ数ベースの**最短経路を保証**
- visitedセットでサイクル回避

### DFS（深さ優先探索）
- **スタック**（または再帰）を使い、深く潜って戻る
- ループ検出・トポロジカルソート・連結成分の列挙に使う

## 計算量

| 操作 | 隣接リスト |
|------|-----------|
| BFS / DFS | O(V + E) |
| 全ノード訪問 | O(V + E) |
| 空間（キュー/スタック） | O(V) |

## コード例（Python）

```python
from collections import deque

graph = {0: [1, 2], 1: [3], 2: [3], 3: []}

def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        print(node, end=" ")
        for nb in graph[node]:
            if nb not in visited:
                visited.add(nb)
                queue.append(nb)

def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    if node in visited:
        return
    visited.add(node)
    print(node, end=" ")
    for nb in graph[node]:
        dfs(graph, nb, visited)
```

## よくある誤解・落とし穴

- **BFSは重み付きグラフで最短経路にならない** → ダイクストラ法が必要
- **再帰DFSはスタックオーバーフロー** → 深さ数千超えなら反復実装に切替
- **visitedを忘れると無限ループ** → サイクルのあるグラフでは必須
- **有向グラフの連結性** → 強連結成分（Tarjan・Kosaraju）と弱連結成分は別概念

## 実務での使いどころ

- **FastAPI**: ルーターやミドルウェアの適用順序をDAGで管理・循環依存を検出
- **Neon（PostgreSQL）**: 外部キー参照チェーン（どのテーブルがどれに依存するか）をグラフで可視化
- **Cloud Run**: マイクロサービスの呼び出し関係をDFSで追跡し、障害の伝播範囲を特定
- 権限・ロール階層、GraphQL のリゾルバ依存解決にも応用可能
