# グラフの基本（表現方法・BFS・DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）からなるデータ構造。ネットワーク経路探索・SNSの友人関係・依存関係解析など、現実世界の「つながり」を表現する。BFS/DFSを理解すると、最短経路・サイクル検出・トポロジカルソートといった実務的アルゴリズムへの応用が開ける。

---

## 仕組みの要点

### グラフの種類

- **有向グラフ（Directed）**: エッジに向きがある（例: フォロー関係）
- **無向グラフ（Undirected）**: エッジに向きがない（例: 友人関係）
- **重み付きグラフ**: エッジに数値（コスト・距離）がある
- **DAG（有向非巡回グラフ）**: サイクルがない有向グラフ。依存関係の表現に使われる

### 表現方法の比較

| 方法 | 空間計算量 | 隣接ノード取得 | エッジ存在確認 | 向いている場面 |
|------|-----------|--------------|--------------|--------------|
| 隣接行列 | O(V²) | O(V) | O(1) | 密なグラフ・エッジ存在を頻繁に確認 |
| 隣接リスト | O(V+E) | O(degree) | O(degree) | 疎なグラフ（実務の多くはこちら） |

```
隣接リスト（例: 0→1, 0→2, 1→3）

graph = {
    0: [1, 2],
    1: [3],
    2: [],
    3: []
}
```

### BFS（幅優先探索）

- キューを使い、近いノードから順に探索
- **最短経路（辺の重みが同じ場合）** に使う
- 探索順: 0 → 1 → 2 → 3（層ごと）

### DFS（深さ優先探索）

- スタック（または再帰）を使い、行けるところまで深く探索
- **サイクル検出・トポロジカルソート・連結成分**に使う
- 探索順: 0 → 1 → 3 → 2（深く潜ってからバックトラック）

---

## 計算量

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| BFS / DFS | O(V+E) | O(V²) |
| 全ノード訪問 | O(V+E) | O(V²) |

V = ノード数、E = エッジ数。疎なグラフ（E << V²）では隣接リストが圧倒的に効率的。

---

## コード例（Python）

```python
from collections import deque

graph = {0: [1, 2], 1: [3], 2: [], 3: []}

def bfs(start):
    visited = set()
    queue = deque([start])
    visited.add(start)
    while queue:
        node = queue.popleft()
        print(node, end=" ")
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

def dfs(node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    print(node, end=" ")
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(neighbor, visited)
```

---

## よくある誤解・落とし穴

- **visited管理の忘れ**: サイクルがあると無限ループ。BFS/DFS ともに必須
- **BFS ≠ 常に最短経路**: 重み付きグラフではダイクストラ法が必要
- **再帰DFSのスタックオーバーフロー**: ノード数が多い場合は明示的スタックで反復実装を使う
- **有向グラフの連結性**: 無向グラフと異なり「強連結成分」の概念が必要（Kosarajuアルゴリズムなど）
- **隣接行列のメモリ**: 100万ノードで10¹²バイト → 実用不可。疎なグラフには隣接リストを使う

---

## 実務での使いどころ

### FastAPI + Neon + Cloud Run スタックとの関連

- **Neonのスキーマ依存関係**: テーブル間の外部キー依存をDAGで表現し、マイグレーション順序をトポロジカルソートで決定
- **APIエンドポイントの依存解析**: マイクロサービス化する際に、エンドポイント間の呼び出し関係をグラフで可視化
- **クエリプランの経路**: PostgreSQLのクエリオプティマイザは内部でグラフ探索を使ってJOIN順序を決定している
- **Cloud Runサービス間の依存**: サービスの起動順序や依存関係をDAGで管理するとデプロイ制御が容易

```sql
-- 依存関係の例: 外部キーで連結されたテーブルをBFS的に辿る
WITH RECURSIVE deps AS (
    SELECT table_name FROM migrations WHERE id = 1
    UNION ALL
    SELECT m.table_name FROM migrations m
    JOIN deps d ON m.depends_on = d.table_name
)
SELECT * FROM deps;
```
