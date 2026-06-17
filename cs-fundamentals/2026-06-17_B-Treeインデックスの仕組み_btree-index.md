# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（Balanced Tree）インデックスはRDBMSの検索を劇的に高速化する中核データ構造。PostgreSQL（Neon含む）のデフォルトインデックスタイプ。
- フルスキャンO(N)を O(log N) の検索に変える
- 等値検索・範囲検索・ORDER BY すべてに対応できる汎用性が強み
- インデックス設計の良し悪しがクエリ速度を数桁変えることがある

## 仕組みの要点

**構造**
- ルートノード → 中間ノード → リーフノード の階層ツリー
- 各ノードはページ単位（PostgreSQLは8KB）で管理・I/Oの最小単位
- リーフノードにキー値 + テーブル行へのポインタ（TID）が格納される
- 全リーフノードは双方向リンクリストで連結 → 範囲検索を線形に走査可能

**検索の流れ**
1. ルートから開始、各ノード内を二分探索
2. キー値に基づいて子ノードへ降下（O(log N)段）
3. リーフでTIDを取得 → ヒープ（実テーブル）にアクセスして行を返す

**挿入とページ分割**
1. 対象リーフを特定して挿入
2. ノードが満杯なら「スプリット」: ノードを二分割し、中央キーを親へ昇格
3. スプリットが連鎖することもある（最悪ルートまで伝播）

**バランス維持**
- スプリット・マージにより常に高さO(log N)を保つ
- 100万行のテーブルでも深さは通常3〜4段（1ノードに多数のキーを格納するため）

## 計算量・パフォーマンス特性

| 操作 | 計算量 |
|------|--------|
| 検索（等値） | O(log N) |
| 挿入 | O(log N)＋スプリットコスト |
| 削除 | O(log N) |
| 範囲検索 | O(log N + K) ※K=結果件数 |

- B+Tree（PostgreSQLの実装）: 中間ノードはキーのみ保持、1ページあたりの分岐数が増えて高さが低くなる
- 書き込みのたびにインデックスも更新されるためINSERT/UPDATEにはオーバーヘッドがある

## コード例（SQL）

```sql
-- 基本的なインデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（カラム順が重要）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- 実行計画で Index Scan を確認
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND created_at >= '2026-01-01';
-- 理想: "Index Scan using idx_orders_user_created"

-- カーソルベースページネーション（OFFSETより効率的）
SELECT * FROM orders
WHERE user_id = 42 AND id > :last_id
ORDER BY id LIMIT 20;
```

## よくある誤解・落とし穴

- **「インデックスを増やすほど良い」は誤り**: INSERT/UPDATE/DELETEのたびに全インデックスが更新される
- **複合インデックスの左端の原則**: `(user_id, created_at)` は `WHERE created_at = ?` 単体には使えない
- **低カーディナリティ列**: boolean や status（数種類の値）はインデックスよりシーケンシャルスキャンが速いケースがある
- **LIKE前方一致以外**: `LIKE '%keyword%'` はB-Treeインデックス不使用（全文検索はGINインデックスへ）
- **OFFSET大値問題**: `OFFSET 10000` はインデックスを使っても先頭から10000件読み飛ばす → カーソルベースに切り替える

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon（PostgreSQL）**: 外部キー・WHERE句の条件カラム・ORDER BYカラムにインデックスを貼る基本を徹底する
- **EXPLAIN ANALYZEの習慣化**: ORMが生成するクエリがSeq Scanになっていないか定期的に確認
- **ページネーション**: APIの一覧エンドポイントはカーソルベース（`WHERE id > last_id`）でインデックスを活用
- **Cloud Run × コネクション**: コールドスタート後のコネクション増加でインデックスキャッシュ（shared_buffers）のウォームアップが必要 → PgBouncerなどプーリングと組み合わせると効果的
- **Neonのブランチ機能**: インデックス追加前後でEXPLAIN結果を別ブランチで比較検証できる
