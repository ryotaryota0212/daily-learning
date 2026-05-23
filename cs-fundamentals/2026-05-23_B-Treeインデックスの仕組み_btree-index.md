# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（バランス木）はリレーショナルDBのインデックスに使われる最重要データ構造。
PostgreSQL（Neon）・MySQL・SQLiteすべてのデフォルトインデックスがB-Tree。
「なぜINDEXを貼ると速くなるのか」「なぜ複合インデックスの順番が重要か」を理解する基盤になる。
ストレージI/Oを最小化するよう設計されており、ディスク指向システムに最適化されている。

---

## 仕組みの要点

### 基本構造

- **ノード** = ディスク上の1ページ（PostgreSQLは8KB）
- 各ノードは複数のキーと子ポインタを持つ（二分木ではなく多分木）
- **次数 t**：各ノードは最低 t-1 個、最大 2t-1 個のキーを持つ
- すべての葉ノードが同じ深さ → 常にバランスが保たれる

```
           [30 | 70]
          /    |    \
    [10|20]  [40|60]  [80|90]
```

### B+ Tree（実際のDBが使うもの）

- B-TreeのVariant。**データは葉ノードのみ**に格納
- 葉ノード同士は連結リストで接続 → **範囲検索が高速**
- 内部ノードはキーのみ（ポインタとして機能）

```
内部ノード: [30 | 70]  ← ルーティングのみ
葉ノード:   [10]->[20]->[30]->[40]...  ← 実データ、連結
```

### 検索の流れ

1. ルートノードをディスクから読み込む（1 I/O）
2. キーを比較して子ノードへ降りる
3. 葉ノードに到達したらデータ取得
4. 深さ = log_t(N) 回のI/Oで完了

### 挿入の流れ

1. 葉ノードを探索
2. キーを挿入
3. ノードが溢れたら（2t-1キー超過）**スプリット**（分割）
4. 中央キーを親に昇格、再帰的にバランス調整

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 検索 | O(log N) | ディスクI/O回数 = 木の深さ |
| 挿入 | O(log N) | スプリット発生時も O(log N) |
| 削除 | O(log N) | マージ・再配分あり |
| 範囲検索 | O(log N + K) | K = 結果件数、葉の連結リストを走査 |

**実際の深さ**：1億件でも深さ4〜5程度（次数が大きいため）
- 次数 t=100 の場合：100万件 → 深さ3、10億件 → 深さ5

---

## コード例（PostgreSQLでの確認）

```sql
-- インデックス作成（デフォルトでB-Tree）
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（順番が重要）
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- 実行計画でIndex Scanを確認
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND created_at > '2026-01-01';
-- → Index Scan using idx_orders_user_date

-- 範囲検索もB+Treeの連結リストで効率的
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE created_at BETWEEN '2026-01-01' AND '2026-03-31';
```

---

## よくある誤解・落とし穴

- **「インデックスは貼れば速くなる」** → INSERT/UPDATE/DELETEのたびにB-Treeの更新コストが発生。書き込みが多いテーブルに貼りすぎると逆効果
- **「複合インデックスは順番が何でもいい」** → `(user_id, created_at)` は `WHERE user_id=X` に使えるが `WHERE created_at=X` 単独では使えない（先頭カラムから順に使う）
- **「NULL値はインデックスに含まれない」** → PostgreSQLはNULLをインデックスに含む（IS NULLもIndex Scan可能）。MySQLは異なる挙動
- **「LIKE検索はインデックスが使える」** → `LIKE 'abc%'` は使えるが `LIKE '%abc'` は使えない（前方一致のみ）
- **「テーブルスキャンより常にインデックスが速い」** → 取得行数が多い場合（全体の数%超）はSeq Scanのほうが速い

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

**Neon（PostgreSQL）でのインデックス設計**

- `user_id` で絞り込むクエリが多い → 必ず `user_id` にインデックス
- `created_at` での範囲絞り込み → `(user_id, created_at)` 複合インデックスが有効
- FastAPIのエンドポイントで遅いクエリが出たら `EXPLAIN ANALYZE` で実行計画確認

```python
# FastAPIでのスロークエリ検出パターン
# Neonのクエリログ or pg_stat_statements を活用
# SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC;
```

- **Cloud Runのコールドスタート対策**としてNeonのコネクションプーリング（PgBouncer）を使う場合、インデックスの恩恵を最大化するためにクエリパターンを統一する
- Neonの`pg_stat_user_indexes`でインデックス使用率を監視し、未使用インデックスは削除してWRITEコストを削減する
