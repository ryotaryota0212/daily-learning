# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（Balanced Tree）インデックスは、PostgreSQL・MySQLなど主要RDBMSのデフォルトインデックス構造。ディスクI/Oを最小化するよう設計された多分岐平衡木で、検索・挿入・削除すべてO(log n)を保証する。
「なぜINDEXを貼るとクエリが速くなるのか」「なぜ複合インデックスの列順が重要なのか」を理解することで、EXPLAINの結果が読め、スロークエリを自分で直せるようになる。
Neonを使ったFastAPI開発では、インデックス設計がスケール時のコスト直結ポイントになる。

---

## 仕組みの要点

### B-Treeの構造

```
                [30 | 70]                ← ルートノード
               /    |    \
        [10|20]  [40|50|60]  [80|90]    ← 内部ノード
        /  |  \      ...       |  \
[1..9][11..19][21..29]    [81..79][91..99]  ← リーフノード（実データへのポインタ）
```

- **ノード** = ディスクの1ページ（通常8KB）に複数のキーとポインタを格納
- **リーフノード** は双方向リンクリストで繋がっている → 範囲検索が速い
- **高さ** = log_m(n)（m=分岐数、通常100〜数百）→ 1億行でも高さ4〜5程度

### 検索の流れ（`WHERE id = 42`）

1. ルートノードをディスクから読み込む
2. 42がどの子に属するかポインタを辿る
3. リーフノードまで降りてヒープ（テーブル本体）のページを取得
4. ディスクI/O回数 ≈ 木の高さ + 1（ヒープアクセス）

### 範囲検索（`WHERE id BETWEEN 10 AND 50`）

- B-Treeで10を見つけたあと、リーフのリンクリストを50まで横断
- フルスキャンと違いランダムI/Oが発生するため、取得行が多すぎるとかえって遅い（オプティマイザがSeqScanを選ぶ閾値は約5〜10%）

### 複合インデックス（`INDEX ON (a, b, c)`）

- キーは左から順に比較される
- `WHERE a = 1 AND b = 2` → 使える
- `WHERE b = 2` → 使えない（左端のaがない）
- `WHERE a = 1 AND c = 3` → aだけ使ってcはフィルタ

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 等価検索 | O(log n) | ページキャッシュ効果で実際はより速い |
| 範囲検索 | O(log n + k) | k=結果件数 |
| 挿入 | O(log n) | ページ分割が発生すると書き込み増加 |
| 削除 | O(log n) | ページマージが発生する場合あり |
| フルスキャン | O(n) | インデックスなし |

**ページ分割のコスト**：挿入でノードが満杯になると分割が発生し、最悪ルートまで伝播する。大量INSERT時のパフォーマンス劣化の主因。

---

## コード例（EXPLAINでインデックス効果を確認）

```sql
-- インデックス作成
CREATE INDEX idx_orders_user_created
  ON orders(user_id, created_at DESC);

-- 実行計画確認
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE user_id = 123
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY created_at DESC
LIMIT 20;

-- 期待する出力キーワード:
-- Index Scan using idx_orders_user_created  ← インデックス利用
-- Buffers: shared hit=3                     ← キャッシュヒット数（ディスクI/O=0）
-- rows=20 (actual rows=20)                  ← 見積もり精度
```

---

## よくある誤解・落とし穴

- **「インデックスを貼れば必ず速くなる」**
  → 取得行が多い場合、Seq Scanの方が速い。オプティマイザが判断するが、統計情報が古いと誤る（`ANALYZE`が必要）

- **「ORDER BYはソートコストがかかる」**
  → インデックスの並び順と一致すれば追加ソートなし。`created_at DESC`のインデックスは降順クエリに直接使える

- **「NULLはインデックスに入らない」**
  → PostgreSQLはNULLもインデックスに格納する（`IS NULL`検索も使える）。MySQLは一部挙動が異なる

- **「複合インデックスは必ず全列使う」**
  → 左前置列のみ使うPartial Scanも有効。ただし右端の列だけでは機能しない

- **インデックスの肥大化**
  → UPDATEが多いテーブルは「dead tuple」でインデックスが膨らむ。`REINDEX`または`autovacuum`で整理

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

**Neonでのインデックス設計パターン：**

```sql
-- ユーザー別の最新N件取得（よくあるパターン）
CREATE INDEX idx_posts_user_created
  ON posts(user_id, created_at DESC);

-- 部分インデックス：アクティブレコードのみ対象にしてサイズ削減
CREATE INDEX idx_active_users
  ON users(email)
  WHERE deleted_at IS NULL;
```

- **Cloud Runのコールドスタート**：接続確立後の最初のクエリが遅い場合、統計情報の更新タイミングを疑う
- **Neonのブランチ機能**：本番と同じスキーマ・データでインデックス変更を検証してから本番適用できる
- **コスト意識**：インデックスは読み取りを速くするが書き込みコストが増える。書き込み多めのテーブルではインデックス数を絞る
- **EXPLAIN ANALYZEの習慣化**：FastAPIのSQLAlchemyから発行されるSQLをログに出して定期確認する

```python
# SQLAlchemyでスロークエリを検出する設定例
engine = create_engine(
    DATABASE_URL,
    echo=False,  # 本番はFalse
    connect_args={"options": "-c log_min_duration_statement=100"}  # 100ms以上をログ
)
```
