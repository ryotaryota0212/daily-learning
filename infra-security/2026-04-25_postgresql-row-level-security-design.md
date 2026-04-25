# PostgreSQL Row Level Security (RLS) 設計パターン

## 概要
RLS は PostgreSQL がテーブルの「行単位」でアクセス制御するネイティブ機能。アプリ層の `WHERE user_id = ?` を**忘れた瞬間に情報漏洩する**問題を、DB 側で強制的に塞ぐための最後の砦になる。マルチテナント SaaS や Firebase Auth でユーザー識別する FastAPI + Neon 構成では、RLS を入れるか否かで「設計が壊れにくいか」が大きく変わる。

## 仕組みの要点
- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` で有効化、`CREATE POLICY` で行フィルタを定義
- ポリシーは `USING`（読み取り/更新の対象行）と `WITH CHECK`（書き込み可否）に分かれる
- 評価コンテキストは `current_setting('app.tenant_id', true)` のようなセッション変数で渡す
- `BYPASSRLS` 属性または `SUPERUSER` はポリシーをスキップする（マイグレーション用ロールにのみ付与）
- `FORCE ROW LEVEL SECURITY` を付けるとテーブル所有者にも適用される（重要）
- Neon のコネクションプーリング（PgBouncer transaction mode）では `SET LOCAL` を必ずトランザクション内で使う

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| アプリ層の `WHERE user_id = ?` だけで制御 | RLS で DB 側にも強制する多層防御 |
| `SET app.tenant_id` をセッション全体で使う（プール汚染） | `SET LOCAL` をトランザクション開始時に毎回 |
| アプリと管理処理を同じ DB ロールで実行 | アプリ用ロール（RLS 適用）と管理ロール（BYPASSRLS）を分離 |
| `ENABLE` だけで `FORCE` を付け忘れる | テーブル所有者でも RLS が効く `FORCE` を付ける |
| ポリシーに重い JOIN を書く | 単純な `=` 比較に絞り、必要なら関数でキャッシュ |

## 設計例（FastAPI + Neon）

```sql
-- マイグレーションは管理ロールで（RLS をバイパス）
CREATE TABLE notes (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL, body text);
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;
ALTER TABLE notes FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON notes
  USING (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

```python
# FastAPI: リクエストごとに Firebase Auth から得た tenant_id を注入
async def db_session(uid: str = Depends(verify_firebase_token)):
    async with engine.begin() as conn:
        await conn.execute(text("SET LOCAL app.tenant_id = :t"), {"t": uid})
        yield conn  # 以降の SELECT/UPDATE は自動で行フィルタされる
```

## トレードオフ

| 観点 | RLS あり | RLS なし（アプリ層のみ） |
|---|---|---|
| 漏洩リスク | DB 側で強制、ヒューマンエラーに強い | `WHERE` 忘れで全件露出 |
| パフォーマンス | ポリシー分のオーバーヘッド（数%〜10%） | 余計なフィルタなし |
| デバッグ | 「行が見えない」原因がポリシーか条件か切り分け必要 | アプリのコードを読めば自明 |
| マイグレーション | BYPASSRLS ロールの分離が必須 | 単純 |
| ORM 互換性 | SET LOCAL の挿入ポイントを設計する必要 | 自由 |

## チェックリスト
- [ ] アプリ用 DB ロールに `BYPASSRLS` を付けていない
- [ ] 全テナント分離テーブルで `ENABLE` + `FORCE` 両方を有効化
- [ ] `SET LOCAL` をトランザクション内で毎回実行（プール汚染防止）
- [ ] ポリシーで使う列にインデックスがある（`tenant_id` 等）
- [ ] 「ポリシーを外したら何件見えるか」をテストで検証している
