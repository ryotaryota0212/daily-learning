# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で構成されるデータ構造で、SNSの友人関係・ルーティング・依存関係解決など現実の問題を自然にモデル化できる。BFS/DFSはグラフ探索の2大アルゴリズムで、最短経路・到達可能性・サイクル検出などの基礎となる。FastAPIのルーティング依存解決やCI/CDのタスク順序決定にも内部で使われる考え方。

---

## 仕組みの要点

### グラフの種類
- **有向グラフ（Directed）**: エッジに向きあり（A→B ≠ B→A）。例：依存関係、フォロー関係
- **無向グラフ（Undirected）**: エッジに向きなし（A–B = B–A）。例：友人関係、道路網
- **重み付きグラフ**: エッジに数値（距離・コスト）あり。最短経路問題で使う
- **DAG（有向非巡回グラフ）**: サイクルなし。タスクスケジューリング・依存解決に頻出

### グラフの表現方法2択

| 表現 | 構造 | メモリ | エッジ確認 | 隣接ノード列挙 |
|------|------|--------|-----------|--------------|
| 隣接リスト | `{node: [neighbors]}` | O(V+E) | O(degree) | O(degree) |
| 隣接行列 | `V×V の2D配列` | O(V²) | O(1) | O(V) |

- **疎グラフ**（エッジ少）→ 隣接リスト一択
- **密グラフ**（エッジ多）→ 隣接行列も検討

### BFS（幅優先探索）の仕組み
1. 開始ノードをキューに入れ、visitedに追加
2. キューから取り出し→隣接ノードを未訪問ならキューへ
3. キューが空になるまで繰り返す
- **特徴**: 近いノードから順に探索 → **最短ホップ数**を保証（重みなしグラフ）

### DFS（深さ優先探索）の仕組み
1. 開始ノードをスタック（または再帰）でたどる
2. 訪問済みをマーク → 隣接ノードを再帰的に探索
3. 行き止まりで戻る（バックトラック）
- **特徴**: 深く潜る → サイクル検出・トポロジカルソート・連結成分の判定

---

## 計算量・パフォーマンス特性

| 操作 | BFS | DFS |
|------|-----|-----|
| 時間計算量 | O(V+E) | O(V+E) |
| 空間計算量 | O(V)（キュー） | O(V)（スタック/再帰深さ） |

- V=頂点数、E=辺数
- 最短経路: BFS（重みなし）/ Dijkstra（重みあり）
- DFSの再帰はグラフが深いとスタックオーバーフローに注意

---

## コード例（Python）

```python
from collections import defaultdict, deque

graph = defaultdict(list)
for u, v in [(0,1),(0,2),(1,3),(2,3),(3,4)]:
    graph[u].append(v)

def bfs(start):
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

def dfs(node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    result = [node]
    for neighbor in graph[node]:
        if neighbor not in visited:
            result += dfs(neighbor, visited)
    return result

print(bfs(0))   # [0, 1, 2, 3, 4]
print(dfs(0))   # [0, 1, 3, 4, 2]
```

---

## よくある誤解・落とし穴

- **BFSは常に最短経路を返す → 重みなしグラフのみ**。重みありでは使えない
- **DFSで最短経路は求められない** → ただしDAGなら工夫次第で可
- **visitedマークのタイミング**: キューに入れる前にマークしないと同じノードを複数回追加してしまう（BFSの典型バグ）
- **無向グラフの隣接リスト**: 双方向に追加しないと片側だけのグラフになる
- **大きなグラフのDFS再帰**: Pythonのデフォルト再帰制限は1000。`sys.setrecursionlimit`か反復DFSに切り替える

---

## 実務での使いどころ

| シーン | 使うアルゴリズム |
|--------|----------------|
| FastAPIの依存関係（Depends）の解決順序 | 内部的にDAGのトポロジカルソート |
| Cloud Runのサービス間の到達可能性確認 | BFS/DFS |
| NeonのDB移行スクリプトの依存順序決定 | トポロジカルソート（DFSベース） |
| APIレート制限でのリトライグラフ | 状態機械＝有向グラフ |
| `pip`/`npm`のパッケージ依存解決 | DAG + トポロジカルソート |

**個人開発への直接的な応用**: FastAPIで複数のDependsが入れ子になっているとき、フレームワーク内部はDAGとして依存グラフを構築し循環検出を行っている。この仕組みを理解していれば「なぜ循環依存でエラーになるか」が直感的にわかる。
