# グラフの基本（表現方法、BFS、DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）から構成されるデータ構造。SNSの友達関係、経路探索（Google Maps）、タスクの依存関係解決（CI/CDパイプライン）など実務のあらゆる場面に登場する。BFS/DFSの使い分けを理解することで、「最短経路」「到達可能か」「循環があるか」といった問題を自力で解けるようになる。

---

## 仕組みの要点

### グラフの種類

- **有向グラフ（Directed）**: エッジに方向あり（例：Twitterのフォロー関係）
- **無向グラフ（Undirected）**: 方向なし（例：Facebookの友達関係）
- **重み付きグラフ**: エッジに重み（距離・コスト）あり
- **DAG（有向非巡回グラフ）**: 閉路のない有向グラフ。依存関係の表現に使う

### 表現方法

**隣接リスト（Adjacency List）**

```
graph = {
  "A": ["B", "C"],
  "B": ["D"],
  "C": ["D", "E"],
  "D": [],
  "E": []
}
```

- メモリ: O(V + E)　← スパースグラフに有利
- 隣接ノード列挙: O(deg(v))

**隣接行列（Adjacency Matrix）**

```
     A  B  C  D  E
A  [ 0, 1, 1, 0, 0 ]
B  [ 0, 0, 0, 1, 0 ]
```

- メモリ: O(V²)　← 密グラフ向き
- エッジ存在確認: O(1)

### BFS（幅優先探索）

キューを使い、近い順に探索する。

- **用途**: 最短経路（重みなし）、到達可能性の確認
- 訪問済み管理を忘れると無限ループになる

```python
from collections import deque

def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph.get(node, []):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return visited
```

### DFS（深さ優先探索）

スタック（または再帰）で深く潜ってから戻る。

- **用途**: 閉路検出、トポロジカルソート、連結成分の列挙
- 再帰の深さに注意（Pythonはデフォルト1000）

```python
def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    for neighbor in graph.get(node, []):
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
    return visited
```

---

## 計算量・パフォーマンス特性

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| BFS/DFS | O(V + E) | O(V²) |
| エッジ存在確認 | O(deg(v)) | O(1) |
| 全エッジ列挙 | O(V + E) | O(V²) |
| メモリ | O(V + E) | O(V²) |

- V = 頂点数、E = 辺数
- 実務のグラフはスパース（E << V²）が多い → 隣接リストが基本

---

## よくある誤解・落とし穴

- **visited管理の抜け**: BFS/DFSで同一ノードを複数回処理し、無限ループや誤集計が起きる
- **BFSで最短経路が保証されるのは「重みなし」のみ**: 重みがあればDijkstra法が必要
- **DFSの再帰深度**: ノード数が多いと`RecursionError`。反復DFS（明示的スタック）に変換する
- **有向グラフの閉路検出**: 単純な`visited`だけでは不十分。「処理中」状態も管理する必要がある

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**

- **依存関係の解決**: Cloud Runサービス間の起動順序管理はDAGとトポロジカルソートで考えられる
- **Neonのスキーマ設計**: テーブル間の外部キー参照関係を有向グラフとして捉えると、正規化や循環参照の問題を整理しやすい
- **APIエンドポイントのフロー分析**: リクエストが複数サービスを経由するとき、到達可能性・循環をグラフで検証できる
- **キャッシュ無効化**: 依存するリソースが更新されたとき、BFSで連鎖的に無効化するキャッシュを特定する

---

*作成日: 2026-05-08*
