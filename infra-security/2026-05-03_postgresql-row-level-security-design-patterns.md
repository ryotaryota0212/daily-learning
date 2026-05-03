# PostgreSQL Row Level Security (RLS) 設計パターン

## 概要

マルチテナントSaaSや個人データを扱うアプリでは「自分のデータしか見えない」を保証することが必須。
アプリ層のWHERE句に依存すると、1箇所のバグで他人のデータが漏れる。
RLSは「DB自身が行単位でアクセス制御する」仕組みで、アプリの実装ミスを最後の砦としてDB側で塞ぐ。
Neon (Postgres) + FastAPI + Firebase Auth スタックでは、FirebaseのUIDをセッション変数経由でDBに渡す設計が定石。

## 仕組みの要点

- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` で行単位のフィルタを有効化
- `CREATE POLICY` で「どの条件の行を SELECT/INSERT/UPDATE/DELETE できるか」を定義
- ポリシーは **USING句**（読める行）と **WITH CHECK句**（書ける行）の2軸で書く
- アプリは接続のたびに `SET LOCAL app.current_user_id = '...'` でユーザーIDを注入
- `BYPASSRLS` 権限を持つロール（マイグレーション用）と、持たないロール（アプリ用）を分ける
- `FORCE ROW LEVEL SECURITY` をテーブル所有者にも適用しないと、所有者は素通りする

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| アプリのWHERE句だけで分離 | DB側のRLSで二重ガード |
| アプリ全体で同じDBユーザー＋管理者権限 | アプリ用ロールは `BYPASSRLS` なし |
| `current_setting('app.uid')` をデフォルト未指定で使う | `current_setting('app.uid', true)` で NULL 許容＋失敗時拒否 |
| ポリシーはUSING句だけ | UPDATE/INSERTには WITH CHECK も必須 |
| マイグレーションも同じ接続 | 管理用ロールは別接続で `BYPASSRLS` |

## 設計例（最小）

```sql
-- アプリ用ロール (BYPASSRLSなし)
CREATE ROLE app_user LOGIN PASSWORD '...';

CREATE TABLE notes (
  id uuid PRIMARY KEY,
  owner_id text NOT NULL,
  body text NOT NULL
);
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;
ALTER TABLE notes FORCE ROW LEVEL SECURITY;

CREATE POLICY notes_owner ON notes
  USING      (owner_id = current_setting('app.uid', true))
  WITH CHECK (owner_id = current_setting('app.uid', true));

GRANT SELECT, INSERT, UPDATE, DELETE ON notes TO app_user;
```

```python
# FastAPI: リクエストごとにUIDをセット (SQLAlchemy)
async def db_session(uid: str = Depends(verify_firebase_token)):
    async with SessionLocal() as s:
        await s.execute(text("SET LOCAL app.uid = :uid"), {"uid": uid})
        yield s
```

## トレードオフ

| 観点 | RLSあり | RLSなし（アプリ層のみ） |
|---|---|---|
| 漏洩耐性 | ◎ DBが最終防衛線 | × 1箇所のミスで全件漏れ |
| パフォーマンス | △ ポリシー評価コスト＋インデックス必須 | ◎ 素のクエリ |
| デバッグ | △ 行が「消えて見える」と原因追跡が難しい | ◎ クエリだけ見れば分かる |
| 管理運用 | △ ロール分離・接続管理が増える | ◎ シンプル |
| バッチ・移行 | △ `BYPASSRLS` 別ロールが必要 | ◎ そのまま流せる |

## チェックリスト

- [ ] アプリ用ロールに `BYPASSRLS` が付いていないか確認した
- [ ] `FORCE ROW LEVEL SECURITY` を有効にした（所有者経由の漏れ防止）
- [ ] `current_setting('app.uid', true)` で未設定時は NULL → ポリシーで拒否される
- [ ] ポリシー対象列（owner_id 等）にインデックスを張った
- [ ] RLSが効いていることを「他人のIDで自分のレコードが見えない」テストで検証した
