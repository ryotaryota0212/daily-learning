# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフは「ノード（頂点）とエッジ（辺）の集合」で、ネットワーク・依存関係・ルート探索など実務の至る所に現れる。SNSの友人関係、マイクロサービスの呼び出し関係、DBのクエリプランも全てグラフで表現できる。BFS・DFSを理解することで「最短経路」「閉路検出」「到達可能性」の問題を自力で解けるようになる。

---

## グラフの表現方法

### 隣接リスト（Adjacency List）

- 各ノードが「隣接ノードのリスト」を持つ
- メモリ: O(V + E)
- 辺の存在確認: O(degree)
- **疎グラフ（辺が少ない）に最適** → 実務でほぼこちらを使う

```
graph = {
    0: [1, 2],
    1: [0, 3],
    2: [0],
    3: [1]
}
```

### 隣接行列（Adjacency Matrix）

- V×Vの2次元配列。`matrix[i][j] = 1` なら辺あり
- メモリ: O(V²)
- 辺の存在確認: O(1)
- **密グラフ（辺が多い）または辺の存在を頻繁に確認する場合**

---

## BFS（幅優先探索）

- **キュー**を使って近い順にノードを探索
- 最短経路（辺に重みがない場合）の保証あり
- 用途: 最短経路、レイヤー別処理、連結成分

```python
from collections import deque

def bfs(graph, start):
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
```

- 計算量: O(V + E)

---

## DFS（深さ優先探索）

- **スタック**（または再帰）を使って奥深くまで探索
- 用途: 閉路検出、トポロジカルソート、連結成分、バックトラック

```python
def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    for neighbor in graph.get(start, []):
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
    return visited
```

- 計算量: O(V + E)
- 再帰が深いとスタックオーバーフローに注意 → 大きなグラフは反復DFSへ

---

## 計算量まとめ

| 操作         | 隣接リスト | 隣接行列 |
|------------|-----------|---------|
| メモリ       | O(V+E)    | O(V²)   |
| BFS/DFS    | O(V+E)    | O(V²)   |
| 辺の存在確認 | O(degree) | O(1)    |

---

## よくある誤解・落とし穴

- **訪問済み管理を忘れる** → 無限ループ（閉路があれば必ず起きる）
- **BFSが最短経路を保証するのは重みなしグラフのみ** → 重みありはダイクストラが必要
- **有向グラフと無向グラフを混同** → 隣接リスト構築時に方向を意識する
- **再帰DFSの深さ制限** → Pythonデフォルトは1000。`sys.setrecursionlimit`で変更可能だが非推奨

---

## 実務での使いどころ

| シーン | 活用法 |
|------|-------|
| FastAPI依存注入 | DIグラフをトポロジカルソートで初期化順を決定 |
| Neon（PostgreSQL） | クエリのJOIN関係をグラフで把握してインデックス設計 |
| Cloud Runサービス間呼び出し | サービス依存グラフからデプロイ順序・障害伝播を分析 |
| タスクスケジューラ | 依存タスクのDAG（有向非巡回グラフ）をトポロジカルソート |

**実装のコツ**: Pythonでは `collections.defaultdict(list)` で隣接リストを簡潔に管理できる。グラフ問題に当たったらまず「有向 or 無向」「重みあり or なし」「閉路あり or DAG」の3点を確認する。
