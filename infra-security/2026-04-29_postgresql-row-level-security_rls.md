# PostgreSQL Row Level Security (RLS) 設計パターン

## 概要

マルチテナントSaaSやユーザーごとのデータ分離が必要な場面で、アプリ層のWHERE句だけに頼ると「1箇所漏らすと全テナントのデータが漏洩」する。RLSはDB側で行レベルのアクセス制御を強制でき、**アプリの実装ミスに対する最後の防衛線**になる。Neon (PostgreSQL) + FastAPI 構成では、接続ごとにテナントIDをセッション変数で渡すパターンが標準的。

## 仕組みの要点

- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` で行単位の制御をON
- `CREATE POLICY` で「どのロール」が「どの行」を「何の操作 (SELECT/INSERT/UPDATE/DELETE)」できるか定義
- ポリシーの判定式に `current_setting('app.tenant_id')` のようなセッション変数を使うのが定番
- `BYPASSRLS` 属性を持つロール（例: マイグレーション用）はRLSを無視できる
- テーブル所有者は**デフォルトでRLSが効かない**点に注意（`FORCE ROW LEVEL SECURITY` で強制）
- `USING` 句 = 既存行の可視性、`WITH CHECK` 句 = 新規/更新行の検証

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| WHERE句でテナント絞り込み（書き忘れで漏洩） | RLSで強制、アプリは tenant_id をセッションに設定するだけ |
| 全クエリでアプリロール=DB所有者 | 所有者と実行ロールを分離し `FORCE RLS` |
| ポリシー式に `IS NULL` 比較を入れる | `coalesce` で明示、未設定時はアクセス拒否 |
| 接続プールでセッション変数が漏れる | リクエスト毎に `SET LOCAL` でトランザクション境界に閉じる |
| 管理画面用に「特権ロール」を共有 | `BYPASSRLS` ロールを別途用意し用途を限定 |

## 設計例（FastAPI + Neon）

```sql
-- 1. ポリシー定義
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents FORCE  ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
  USING      (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

```python
# 2. FastAPI: リクエスト毎にトランザクション内で SET LOCAL
from sqlalchemy import text

async def db_session(request: Request):
    async with AsyncSessionLocal() as s:
        async with s.begin():
            await s.execute(
                text("SET LOCAL app.tenant_id = :t"),
                {"t": str(request.state.tenant_id)},
            )
            yield s  # commit はここで自動
```

ポイント: `SET LOCAL` はトランザクション終了で消えるので、コネクションプール経由で**他リクエストに漏れない**。

## トレードオフ

| 観点 | RLS有り | RLS無し（アプリ層のみ） |
|---|---|---|
| セキュリティ | 漏洩耐性が高い（多層防御） | 1箇所のバグで全件漏洩 |
| パフォーマンス | ポリシー式がプラン化されるためインデックス必須 | 素のクエリ、最適化しやすい |
| デバッグ容易性 | 「行が見えない」原因がDB層にあり初見で詰まる | アプリログだけで追える |
| マイグレーション | `BYPASSRLS` ロール運用が必要 | 通常運用 |
| ORM互換 | `SET LOCAL` の挿し込みが必要 | そのまま |

## チェックリスト

- [ ] `ENABLE` だけでなく `FORCE ROW LEVEL SECURITY` も入れたか（所有者バイパス防止）
- [ ] ポリシーは `USING` と `WITH CHECK` の両方を定義したか（INSERT/UPDATE経路の漏れ防止）
- [ ] テナントIDは `SET LOCAL` でトランザクション内に閉じているか（プール越え漏洩防止）
- [ ] ポリシー式で参照する列に複合インデックス `(tenant_id, ...)` が貼ってあるか
- [ ] 「テナント未設定時はアクセス拒否」になることをE2Eテストで確認したか
