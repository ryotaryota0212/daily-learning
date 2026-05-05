# PostgreSQL Row Level Security（RLS）設計パターン — Neon × FastAPI 前提

## 概要

RLS は「行単位のアクセス制御を DB 側で強制する」機能。
アプリ側の `WHERE user_id = ?` を書き忘れた瞬間に他人のデータを返す事故を、**DB が最後の防衛線として防ぐ**。
マルチテナント SaaS や Firebase Auth で発行された UID で行を分離するシステムでは、RLS が「設計を成立させる土台」になる。
アプリのバグやクエリ改竄に対して**多層防御**を効かせるための基本装備として理解しておきたい。

---

## 仕組みの要点

- テーブルごとに `ENABLE ROW LEVEL SECURITY` を有効化し、`POLICY` で可視/書込ルールを定義
- ポリシーはセッション変数（`current_setting('app.user_id')`）や DB ロールを参照可能
- アプリは「リクエストごとに `SET LOCAL app.user_id = '...'`」を発行 → トランザクション終了で自動失効
- `BYPASSRLS` 属性を持つロール（管理用）以外は必ずポリシーを通る
- `USING`（読み取り条件）と `WITH CHECK`（書き込み条件）を分けて指定できる
- スーパーユーザーや所有者は既定で素通りするので、**アプリ用ロールは別に作成**する

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| アプリの WHERE 句だけでテナント分離 | RLS で DB 側にも強制 |
| アプリが管理者ロールで全クエリ実行 | 最小権限ロール＋ `SET LOCAL` でユーザー文脈を渡す |
| ポリシーを `USING` だけ書いて INSERT を放置 | `WITH CHECK` で書込側も縛る |
| `current_user` を直接ポリシーに使う（接続プールで誤動作） | `current_setting('app.user_id', true)` を使う |
| マイグレーションで RLS 無効のままデプロイ | CI で `relrowsecurity` を検査 |

---

## 設計例（FastAPI + Neon）

```sql
-- 1. ロール分離とテーブル設定
CREATE ROLE app_user NOINHERIT LOGIN PASSWORD '...';
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;
ALTER TABLE notes FORCE ROW LEVEL SECURITY;  -- 所有者にも適用

CREATE POLICY notes_isolation ON notes
  USING (user_id = current_setting('app.user_id', true))
  WITH CHECK (user_id = current_setting('app.user_id', true));

GRANT SELECT, INSERT, UPDATE, DELETE ON notes TO app_user;
```

```python
# 2. FastAPI 依存関数で毎リクエスト user_id を注入
async def db_session(uid: str = Depends(verify_firebase_token)):
    async with engine.begin() as conn:
        await conn.execute(text("SET LOCAL app.user_id = :u"), {"u": uid})
        yield conn  # トランザクション終了で SET LOCAL は自動失効
```

ポイント: `SET LOCAL` はトランザクション内のみ有効。コネクションプール経由でも他リクエストに漏れない。

---

## トレードオフ

| 観点 | RLS あり | アプリ層のみ |
|---|---|---|
| 安全性 | バグ・SQL 改竄に強い | WHERE 抜けで即漏洩 |
| パフォーマンス | ポリシー評価コスト（インデックスで吸収可） | 純粋なクエリだけ |
| 開発速度 | 初期設計コスト＋デバッグが少し難しい | 速いが事故率高 |
| 監査・コンプライアンス | 説明しやすい（DB に証跡） | 説明責任がアプリに集中 |
| BI / 集計クエリ | 管理ロールで `BYPASSRLS` 必要 | 自由 |

トレードオフの本質: **「速さ」と「壊れにくさ」のどちらを取るか**。  
ユーザーデータを扱うなら RLS の初期コストは保険料として安い。

---

## よくある落とし穴

- `current_setting('app.user_id')` を引数なしで呼ぶと未設定時に例外 → `true` を第二引数に
- マイグレーションツール（Alembic 等）が DB 所有者で動く → `FORCE ROW LEVEL SECURITY` を忘れない
- `EXPLAIN` でポリシー条件がインデックスに乗っているか必ず確認（フルスキャン化に注意）
- PgBouncer の transaction pooling 下では `SET`（セッション）ではなく **必ず `SET LOCAL`** を使う
- テスト時に「RLS が効いた状態」と「管理者で全件見える状態」を両方再現できるフィクスチャを用意

---

## チェックリスト

- [ ] アプリ用ロールに `BYPASSRLS` が付いていないか
- [ ] 全マルチテナントテーブルで `ENABLE` ＋ `FORCE ROW LEVEL SECURITY`
- [ ] `USING` と `WITH CHECK` の両方を定義したか
- [ ] リクエスト毎に `SET LOCAL app.user_id` をトランザクション開始時に実行しているか
- [ ] ポリシー条件カラム（`user_id` / `tenant_id`）にインデックスがあるか
