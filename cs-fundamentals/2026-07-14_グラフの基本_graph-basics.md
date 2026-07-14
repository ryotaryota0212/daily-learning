# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で構成されるデータ構造で、現実世界の「関係性」を表現する最も汎用的なモデル。SNSのフォロー関係、道路ネットワーク、依存関係解決（npmパッケージ等）、APIのルーティングなど、実務の至る所に登場する。BFS/DFSの使い分けを理解することで「最短経路か、到達可能性か」という問いに即座に答えられるようになる。

---

## 仕組みの要点

### グラフの種類

- **有向グラフ（Directed）**: エッジに向きあり（例: フォロー関係 A→B ≠ B→A）
- **無向グラフ（Undirected）**: エッジに向きなし（例: 友人関係）
- **重み付きグラフ（Weighted）**: エッジにコストあり（例: 距離、レイテンシ）
- **DAG（有向非巡回グラフ）**: サイクルなし → トポロジカルソートが可能

### 表現方法

**隣接リスト（Adjacency List）**
- 各ノードの隣接ノードをリストで保持
- メモリ: O(V + E)、スパースグラフに有利
- エッジ確認: O(degree)

**隣接行列（Adjacency Matrix）**
- V×V の2次元配列
- メモリ: O(V²)、密グラフ向き
- エッジ確認: O(1)

```
隣接リスト例:
A -> [B, C]
B -> [D]
C -> [D, E]
D -> []
E -> []
```

### BFS（幅優先探索）

- キューを使って「近いノードから順に」探索
- **最短ホップ数**を保証（重みなしグラフ）
- 到達可能な全ノードを距離順に列挙

```
BFS動作（A起点）:
Queue: [A]
訪問: A → Queue: [B, C]
訪問: B → Queue: [C, D]
訪問: C → Queue: [D, E]
...
```

### DFS（深さ優先探索）

- スタック（または再帰）を使って「奥まで潜ってから戻る」探索
- **到達可能性の確認**、サイクル検出、トポロジカルソートに適する
- メモリ使用量はBFSより少ない（枝の深さ分のスタック）

---

## 計算量・パフォーマンス特性

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| BFS/DFS | O(V + E) | O(V²) |
| エッジ存在確認 | O(degree) | O(1) |
| メモリ | O(V + E) | O(V²) |

- スパースグラフ（E << V²）では隣接リストが圧倒的に有利
- BFS/DFSは共に O(V + E)。差は「何を解きたいか」にある

---

## コード例（Python）

```python
from collections import deque

graph = {
    "A": ["B", "C"],
    "B": ["D"],
    "C": ["D", "E"],
    "D": [],
    "E": [],
}

def bfs(graph, start):
    visited, queue = {start}, deque([start])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order

def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    order = [start]
    for neighbor in graph[start]:
        if neighbor not in visited:
            order += dfs(graph, neighbor, visited)
    return order

print(bfs(graph, "A"))  # ['A', 'B', 'C', 'D', 'E']
print(dfs(graph, "A"))  # ['A', 'B', 'D', 'C', 'E']
```

---

## よくある誤解・落とし穴

- **「BFSは常に最短経路を返す」は重み付きグラフでは誤り** → 重みありはダイクストラ法が必要
- **訪問済み管理を忘れると無限ループ** → 特に無向グラフやサイクルありのグラフで必須
- **DFSの再帰は深いグラフでスタックオーバーフロー** → Pythonのデフォルト再帰上限は1000。深さが深い場合はスタックで反復実装
- **隣接行列をスパースグラフに使うとメモリ爆発** → 10万ノードなら 10^10 要素 → 非現実的

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **依存関係の解決**: マイクロサービス間の呼び出し順序をDAGで管理 → トポロジカルソートでデプロイ順を決定
- **APIルーティング**: FastAPIのルート解決内部はグラフ構造ではないが、権限チェックのフロー設計にDAGが使える
- **Neonのスキーマ依存**: テーブル間の外部キー関係はグラフ → マイグレーション順序の決定にトポロジカルソート
- **Cloud Runのサービス依存**: 複数サービスの起動順序管理にBFS（幅広く依存を解決）が有効
- **クローリング/データ収集**: ページ間リンクをグラフとして表現し、BFSで特定深さまで収集
