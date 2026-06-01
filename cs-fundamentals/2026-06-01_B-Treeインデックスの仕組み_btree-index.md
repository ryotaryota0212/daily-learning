# B-Treeインデックスの仕組み

## 概要

B-TreeはほぼすべてのRDBMS（PostgreSQL、MySQL、SQLiteなど）がデフォルトで使うインデックス構造。
`WHERE id = 5` や `ORDER BY created_at` のクエリが速い理由はB-Treeにある。
仕組みを知ることで、「なぜこのクエリが遅いのか」「どこにインデックスを張るべきか」を自分で判断できる。

---

## 仕組みの要点

### 木構造のイメージ

```
         [30 | 70]              ← ルートノード
        /    |    \
   [10|20] [40|60] [80|90]     ← 内部ノード
   ↓↓↓     ↓↓↓     ↓↓↓
  実データへのポインタ           ← リーフノード
```

- **ノード** = ディスクのページ（通常8KB）。複数のキーとポインタを格納
- **リーフノード** = 実際の行へのポインタ（ヒープタプルへの参照）を持つ
- **リーフ同士は双方向リンク** → 範囲検索（BETWEEN, >=）が効率的
- 木の高さは非常に浅い。100万行でも高さ3〜4程度

### 検索の流れ（`WHERE id = 42`）

1. ルートノードをディスクから読み込む
2. キーと比較して適切な子ポインタを辿る
3. リーフノードに到達 → ヒープページのポインタを取得
4. 実データのページを読む（ヒープアクセス）

### 挿入の流れ

- リーフノードに空きあり → そこに挿入（O(log n)）
- リーフが満杯 → **ノード分割**（スプリット）が発生
  - ページを2つに分け、中間キーを親に昇格
  - 親も満杯なら再帰的にスプリット

### B-Tree vs B+Tree

- PostgreSQLが使うのは **B+Tree**（厳密にはB-Treeと呼ぶ慣習）
- 内部ノードはキーとポインタのみ、実データポインタはリーフのみ
- → ノードに多くのキーが入る → 木が浅くなる → I/Oが少ない

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 検索（等値） | O(log n) | ディスクI/O回数 = 木の高さ |
| 検索（範囲） | O(log n + k) | k = 取得行数 |
| 挿入 | O(log n) | スプリット時は定数倍の追加コスト |
| 削除 | O(log n) | アンダーフロー時はマージまたは再配置 |
| シーケンシャルスキャン | 不向き | フルスキャンは全ページ読み込みが速い場合も |

- **分岐数（order）** が大きい → 木が浅い → I/O少ない（PostgreSQLのデフォルトは約340）
- キャッシュに乗る上位ノードはメモリから読める → 実質2〜3回のディスクI/Oで済む

---

## コード例（PostgreSQL / SQL）

```sql
-- インデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（左端のカラムから使われる）
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- 実行計画の確認
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 1 AND created_at >= '2026-01-01';

-- インデックス使用確認ポイント
-- "Index Scan" → B-Treeを使っている
-- "Seq Scan"   → フルスキャン（インデックスが効いていない）
-- "Bitmap Index Scan" → 複数インデックスをORで組み合わせ
```

---

## よくある誤解・落とし穴

- **関数をかけるとインデックスが効かない**
  - `WHERE LOWER(email) = 'foo'` → シーケンシャルスキャン
  - 対策: `CREATE INDEX ON users(LOWER(email))` で関数インデックスを作る

- **複合インデックスは左端から使う**
  - `(user_id, created_at)` のインデックスは `WHERE created_at = X` だけでは使われない
  - `WHERE user_id = X` または `WHERE user_id = X AND created_at = Y` なら使われる

- **NULLはインデックスに含まれる（PostgreSQL）**
  - PostgreSQLはNULLをB-Treeに格納する（MySQLは格納しない）

- **カーディナリティが低いと効果薄**
  - `WHERE status IN ('active', 'inactive')` → 全行の50%取得なら全表スキャンの方が速い場合も

- **書き込み負荷が増える**
  - インデックスが多いほどINSERT/UPDATE/DELETEが遅くなる（各インデックスも更新が必要）

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon = PostgreSQLなのでB-Treeインデックスがそのまま使える**

- **EXPLAIN ANALYZEを習慣化する**
  - FastAPIのエンドポイントが遅いとき、まずSQLの実行計画を確認
  - "Seq Scan" + "rows=大量" が見えたらインデックス不足のサイン

- **よく使うパターン**:
  - 外部キー列には必ずインデックス（`user_id`, `post_id` など）
  - ページネーションの `ORDER BY created_at DESC` にインデックス
  - `WHERE is_deleted = false` → カーディナリティ低いので部分インデックスが有効

  ```sql
  -- 部分インデックス（削除済みを除外）
  CREATE INDEX idx_posts_active ON posts(created_at)
  WHERE is_deleted = false;
  ```

- **Cloud Runのコールドスタートとコネクション**: インデックスが効いていればクエリ時間が短縮され、コネクション保持時間も短くできる
