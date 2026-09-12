# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で関係を表すデータ構造。SNSのフォロー関係・経路探索・依存関係解決など実務で頻出する。BFS/DFSはグラフ探索の基本アルゴリズムであり、最短経路・到達可能性・トポロジカルソートの土台になる。

---

## 仕組みの要点

### グラフの種類

- **有向グラフ（Directed）**: A→B は A から B への一方通行。依存関係・タスクフローに使う
- **無向グラフ（Undirected）**: A-B は双方向。友人関係・道路網など
- **重み付きグラフ（Weighted）**: 辺にコストを持つ。最短経路計算に使う
- **DAG（有向非巡回グラフ）**: サイクルなし。タスクスケジューリング・ビルド依存に多用

### グラフの表現方法

| 方法 | 空間計算量 | 隣接頂点の探索 | 辺の存在確認 |
|------|-----------|-------------|------------|
| 隣接リスト | O(V+E) | O(degree) | O(degree) |
| 隣接行列 | O(V²) | O(V) | O(1) |

- **隣接リスト**: 疎グラフ（辺が少ない）に向く。実務ではほぼこちら
- **隣接行列**: 密グラフや「辺が存在するか」を頻繁に問うケースに向く

### BFS（幅優先探索）

- キューを使い、近い頂点から順に探索
- **最短ホップ数**（重みなし）を求める
- 手順: 始点をキューに入れ → 取り出して隣接頂点を未訪問なら追加 → 繰り返す

### DFS（深さ優先探索）

- スタック（または再帰）を使い、深く潜ってから戻る
- **到達可能性・サイクル検出・トポロジカルソート**に使う
- 再帰で書くと簡潔だがスタックオーバーフローに注意（深いグラフでは反復実装へ）

---

## 計算量・パフォーマンス特性

| 操作 | 時間計算量 | 空間計算量 |
|------|-----------|-----------|
| BFS | O(V+E) | O(V) |
| DFS | O(V+E) | O(V) |

- V: 頂点数、E: 辺数
- 隣接リスト実装での値。隣接行列では O(V²) になる

---

## コード例（Python）

```python
from collections import deque

def bfs(graph: dict, start: int) -> list[int]:
    visited = set([start])
    queue = deque([start])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph.get(node, []):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order

def dfs(graph: dict, start: int) -> list[int]:
    visited = set()
    order = []
    def _dfs(node):
        visited.add(node)
        order.append(node)
        for neighbor in graph.get(node, []):
            if neighbor not in visited:
                _dfs(neighbor)
    _dfs(start)
    return order

graph = {0: [1, 2], 1: [3], 2: [3], 3: []}
print(bfs(graph, 0))  # [0, 1, 2, 3]
print(dfs(graph, 0))  # [0, 1, 3, 2]
```

---

## よくある誤解・落とし穴

- **BFSは最短経路を保証する、ただし重みなしグラフのみ**。重み付きグラフにはダイクストラを使う
- **DFSの再帰版は入力が大きいとスタックオーバーフロー**。Pythonのデフォルト再帰制限は1000。`sys.setrecursionlimit()` か反復版に切り替える
- **訪問済みフラグを忘れるとサイクルのあるグラフで無限ループ**になる（有向グラフでも無向グラフでも必須）
- **隣接行列を疎グラフに使うと O(V²) の空間を浪費**。V=100万ならメモリが持たない

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **タスク依存関係（DAG）**: バックグラウンドジョブのステップ管理。タスクAが完了したらタスクBを実行、といった依存を DAG + トポロジカルソートで管理できる
- **関連コンテンツの推薦**: PostgreSQL（Neon）に保存したユーザー行動データを隣接リスト的に扱い、BFSで「N度のつながり」を求める
- **循環依存の検出**: FastAPIルーターやモジュール依存をグラフで表現し、DFSのサイクル検出でビルド時に問題を発見する
- **最短経路**: ルーティング・サイトマップ生成など、リンク構造をグラフで持つと BFS で最短クリック数を計算できる
