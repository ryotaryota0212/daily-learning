# PostgreSQL Row Level Security (RLS) 設計パターン

## 概要

マルチテナント／ユーザー別データ分離を**アプリ層のWHERE句で頑張る**のは事故が起きやすい。1つの `WHERE tenant_id = ?` の付け忘れで全テナントのデータが漏れる。RLS は DB 側で「行単位の可視性」を強制でき、アプリのバグに対する**最後の砦**になる。Neon のような Postgres マネージドでも標準で使え、FastAPI + Firebase Auth スタックと相性が良い。

## 仕組みの要点

- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` で対象テーブルを保護モードにする
- `CREATE POLICY` で「どの行が見えるか／更新できるか」を SQL 式で定義
- セッション変数 (`SET LOCAL app.current_user_id = ...`) を使い、リクエスト単位でユーザー識別子を DB に渡す
- `BYPASSRLS` 属性のあるロール（マイグレーションや管理ジョブ用）と、アプリ用の通常ロールを分ける
- ポリシーは `SELECT` / `INSERT` / `UPDATE` / `DELETE` ごとに `USING`（読み取り条件）と `WITH CHECK`（書き込み条件）で制御

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| アプリ層の WHERE 句だけで分離 | RLS で DB 側でも強制 |
| `superuser` でアプリが接続 | 専用ロール + 最小権限 |
| `current_user_id` をアプリ変数で持つだけ | `SET LOCAL` で DB セッションに紐付け |
| ポリシーを「全部 true」で逃げる | テスト環境でも本番同等のポリシー |
| マイグレーションも RLS 経由 | 管理用は `BYPASSRLS` ロール |

## コード/設計例

```sql
-- ロール分離
CREATE ROLE app_user NOINHERIT LOGIN PASSWORD '...';
CREATE ROLE migrator BYPASSRLS LOGIN PASSWORD '...';

-- RLS 有効化
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;

CREATE POLICY notes_isolation ON notes
  USING      (user_id = current_setting('app.user_id')::uuid)
  WITH CHECK (user_id = current_setting('app.user_id')::uuid);
```

```python
# FastAPI: リクエストごとに user_id を DB セッションに注入
async def with_rls(session, user_id: str):
    await session.execute(
        text("SET LOCAL app.user_id = :uid"), {"uid": user_id}
    )  # トランザクション終了で自動解除
```

Firebase Auth で検証した `uid` を `verify_id_token` 後にこの関数へ渡し、その後の全クエリは RLS で自動フィルタされる。

## トレードオフ

| 観点 | RLS あり | RLS なし（アプリ層分離） |
|---|---|---|
| 安全性 | DB 側で強制、付け忘れに強い | WHERE 句漏れで漏洩リスク |
| 性能 | ポリシー条件が全クエリに付与される | 直接的だが手動 |
| デバッグ | 「なぜ行が見えないか」が分かりにくい | クエリが明示的 |
| 既存コード | 段階導入しやすい | 変更不要 |
| 接続プール | `SET LOCAL` 必須でトランザクション必要 | 制約なし |

## 落とし穴

- **接続プーリング**: PgBouncer の transaction モードでは `SET LOCAL` がトランザクション境界で消えるので OK。session モードは要注意
- **JOIN 先のテーブル**: ポリシー未設定だとそこから漏れる。関連テーブルにも一貫して適用
- **集計クエリ**: `COUNT(*)` も RLS が効くため、想定より少なく見える事象が「仕様通り」
- **Neon の branch**: ブランチごとに独立した Postgres なので、ポリシーもマイグレーションで管理

## チェックリスト

- [ ] アプリ用ロールは `BYPASSRLS` を持たないか
- [ ] テナント／ユーザー識別子は `SET LOCAL` で必ず注入されているか
- [ ] `SELECT` だけでなく `INSERT/UPDATE/DELETE` の `WITH CHECK` も書いたか
- [ ] 関連テーブル（JOIN 先）にもポリシーがあるか
- [ ] CI に「ポリシー無しテーブル検出」テストを入れたか
