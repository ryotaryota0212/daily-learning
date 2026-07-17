# グラフの基本（表現方法、BFS、DFS）

## 概要

グラフはノード（頂点）とエッジ（辺）で構成されるデータ構造。ネットワーク経路探索、SNSの友達関係、依存関係解決など現実の問題を自然にモデル化できる。BFS/DFSは最も基本的な探索アルゴリズムで、より高度なアルゴリズム（Dijkstra、トポロジカルソート）の土台になる。

---

## 仕組みの要点

### グラフの種類
- **有向グラフ (Directed)**: エッジに方向あり。依存関係、URLリンク
- **無向グラフ (Undirected)**: エッジに方向なし。SNSの友達関係
- **重み付きグラフ**: エッジにコストあり。経路の距離・時間

### 表現方法の比較

| 方式 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| メモリ | O(V + E) | O(V²) |
| 辺の検索 | O(degree) | O(1) |
| 向いている場面 | 疎なグラフ（実務の大半） | 密なグラフ |

- **隣接リスト**: `{A: [B, C], B: [D]}` — メモリ効率が良く実務で主流
- **隣接行列**: `matrix[i][j] = 1` — 辺の有無を O(1) で確認できるが疎グラフでは無駄

### BFS（幅優先探索）
- キューを使い、近い頂点から順に探索
- **最短経路（重みなし）を保証**する
- 手順: 開始ノードをキューに追加 → 取り出してとなりをキューに追加 → 訪問済みを管理

### DFS（深さ優先探索）
- スタック（または再帰）を使い、深く潜ってから戻る
- サイクル検出・トポロジカルソート・連結成分の検出に有効
- 手順: 開始ノードを訪問 → 未訪問の隣接ノードへ再帰 → バックトラック

---

## 計算量・パフォーマンス特性

| 操作 | 隣接リスト | 隣接行列 |
|------|-----------|---------|
| BFS / DFS | O(V + E) | O(V²) |
| 辺の追加 | O(1) | O(1) |
| 辺の削除 | O(degree) | O(1) |

- V = 頂点数、E = 辺数
- 疎なグラフ（E << V²）では隣接リストが圧倒的に有利

---

## コード例（Python）

```python
from collections import deque

graph = {
    "A": ["B", "C"],
    "B": ["D", "E"],
    "C": ["F"],
    "D": [], "E": [], "F": []
}

def bfs(graph, start):
    visited, queue = set([start]), deque([start])
    result = []
    while queue:
        node = queue.popleft()
        result.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return result

def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    result = [node]
    for neighbor in graph[node]:
        if neighbor not in visited:
            result += dfs(graph, neighbor, visited)
    return result

print(bfs(graph, "A"))  # ['A', 'B', 'C', 'D', 'E', 'F']
print(dfs(graph, "A"))  # ['A', 'B', 'D', 'E', 'C', 'F']
```

---

## よくある誤解・落とし穴

- **訪問済み管理を忘れる**: サイクルがあると無限ループ。必ず `visited` セットを使う
- **BFS = 最短経路と思いすぎる**: 重み付きグラフでは最短経路にならない（Dijkstra が必要）
- **再帰DFSのスタックオーバーフロー**: 深いグラフでは Python のデフォルト再帰上限（1000）を超えることがある。明示的スタックで反復DFSに変換する
- **隣接行列を大規模グラフに使う**: V=100万なら行列は 10¹² 要素 → メモリ不足

---

## 実務での使いどころ

**FastAPI + Neon + Cloud Run スタックとの関連:**

- **依存関係解決**: マイクロサービスの起動順序やタスクの依存関係をDFSのトポロジカルソートで解決
- **API のルート探索**: FastAPI でリダイレクトチェーンや権限継承ツリーをBFSで辿る
- **Neon のスキーマ分析**: テーブルの外部キー関係をグラフとして表現し、循環参照（循環依存）をDFSで検出
- **Cloud Run サービスグラフ**: サービス間の呼び出し関係を有向グラフでモデル化し、到達可能性やボトルネックを分析
- **最短パス応用**: ユーザー間の「何次の繋がり」を計算する機能はBFSで実装できる
