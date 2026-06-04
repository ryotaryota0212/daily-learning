# B-Treeインデックスの仕組みと検索・挿入の計算量

## 概要

B-Tree（Balanced Tree）は、ディスクI/Oを最小化するために設計された多分木構造。
PostgreSQL・MySQL・SQLiteなど主要RDBMSのデフォルトインデックス実装。
「なぜSELECTが速いのか」「なぜINSERTが遅くなるのか」を理解する鍵。
FastAPI + Neon構成でも、クエリのボトルネックはほぼここに起因する。

---

## 仕組みの要点

**ノード構造**
- ルートノード → ブランチノード → リーフノード の階層構造
- 各ノードは複数のキーと子ポインタを持つ（通常1ノード = 8KBページ）
- リーフノード同士は双方向リンクリストで接続 → 範囲検索が効率的

**バランス維持のメカニズム**
- 挿入時にノードが満杯になると「分割（split）」が発生
- 削除時に不足すると「マージ（merge）」または「再分配」が発生
- 常に全リーフが同じ深さ → 最悪計算量も保証される

**検索の流れ**
1. ルートから開始、キーと比較しながら子ノードへ降りる
2. O(log N)回のノード読み込みでリーフに到達
3. 範囲検索はリーフのリンクリストを辿る

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 点検索 | O(log N) | ツリーの高さに比例 |
| 挿入 | O(log N) | 分割が起きると上位ノードも更新 |
| 削除 | O(log N) | マージ・再分配あり |
| 範囲検索 | O(log N + K) | Kは結果件数 |

- 100万行のテーブルでも高さは約20程度（ディスクアクセス20回以下）
- インデックスが増えるほどINSERT/UPDATEのコストが上がる

---

## コード例（PostgreSQL / Neon）

```sql
-- インデックスなしはSeq Scan（フルスキャン）
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'a@example.com';

-- B-Treeインデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 作成後はIndex Scanに切り替わる
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'a@example.com';

-- 複合インデックス（列順が重要）
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
-- OK: user_id単独、または(user_id + created_at)の検索
-- NG: created_at単独（左端プレフィックスの法則）

-- 関数インデックス（WHERE LOWER(email) の場合）
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

---

## よくある誤解・落とし穴

- **複合インデックスの列順を間違える** → 左端プレフィックスでない列はインデックス不使用
- **LIKE '%keyword'は効かない** → 前方一致（`keyword%`）のみB-Treeで検索可能
- **関数をかけるとインデックス無効** → `WHERE LOWER(email) = ...` は素のインデックスを使えない。関数インデックスで解決
- **インデックスが多すぎる** → 読み取りは速くなるが書き込みは全インデックスを更新するため遅くなる
- **カーディナリティが低い列** → `is_deleted`（trueかfalseのみ）へのインデックスは逆効果なことが多い

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **NeonはPostgreSQLなのでB-Treeがデフォルト**。`CREATE INDEX`で即座に恩恵を受けられる
- FastAPIのエンドポイントでフィルタ・ソートするカラムには必ずインデックスを検討する
- `EXPLAIN ANALYZE`を習慣化する。`Seq Scan`が出たら要注意
- Cloud RunはステートレスなのでDB接続数が爆発しやすい。インデックス最適化で1クエリの実行時間を短縮し、コネクション占有時間を減らす
- NeonのBranching機能でインデックス追加の影響をブランチDBで先に検証できる
