# CI/CDパイプラインのセキュリティと設計

## 概要
CI/CDパイプラインはコードをプロダクションへ届ける自動化の中枢。ここが壊れると「デプロイできない」だけでなく、悪意ある変更が本番に紛れ込む経路にもなる。AI時代にコードの量が増えるほど、パイプライン自体のセキュリティと信頼性設計が重要度を増す。

---

## 仕組みの要点

### パイプラインの典型構成（FastAPI + Cloud Run スタック）
```
push → GitHub Actions → テスト → Dockerビルド → Artifact Registry → Cloud Run デプロイ
```

- **ソース**: GitHubへのpushがトリガー
- **CI（継続的インテグレーション）**: テスト・Lint・セキュリティスキャン
- **CD（継続的デリバリー）**: イメージビルド → レジストリpush → Cloud Runへデプロイ
- **シークレット**: GitHub Actions Secrets / GCP Secret Manager で管理

### セキュリティの主要な関心事
- **サプライチェーン攻撃**: 依存ライブラリやActionsのハイジャック
- **シークレット漏洩**: ログへの埋め込み、環境変数の誤設定
- **権限過多**: パイプラインが本番DBに直接アクセスできる状態
- **イメージの改ざん**: 署名なしイメージのデプロイ

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `actions/checkout@main` (最新固定) | `actions/checkout@v4.1.1` (SHA固定が理想) |
| シークレットをログ出力・`echo`で確認 | シークレットは絶対にprintしない |
| 全ブランチが本番へデプロイ | `main`ブランチのみ本番CDが走る |
| CI用SAが`roles/editor`を持つ | 最小権限SA（Artifact Registry書込み + Cloud Run Admin のみ） |
| ベースイメージを`python:latest`で固定なし | `python:3.12.3-slim` のようにバージョン固定 |
| テストなしでデプロイ | テスト失敗時はデプロイブロック |

---

## 設計例（GitHub Actions ワークフロー）

```yaml
name: CI/CD
on:
  push:
    branches: [main]

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Workload Identity用

    steps:
      - uses: actions/checkout@v4.1.1

      - name: Run tests
        run: |
          pip install -r requirements.txt
          pytest --tb=short

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2.1.3
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.SA_EMAIL }}

      - name: Build and push image
        run: |
          docker build -t $IMAGE_TAG .
          docker push $IMAGE_TAG

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy myapp --image=$IMAGE_TAG --region=asia-northeast1
```

**ポイント**: `id-token: write` + Workload Identity Federation でサービスアカウントキー不要。

---

## トレードオフ

| 観点 | シンプル構成 | セキュア構成 |
|---|---|---|
| SAキー | JSONキーをシークレットに保存 | Workload Identity Federation（キーなし） |
| Actionsバージョン | `@main` で常に最新 | SHA固定で改ざん検知 |
| デプロイ戦略 | 直接デプロイ（ダウンタイムあり） | Blue/Green or ローリング更新 |
| スキャン | なし | Trivy等でイメージ脆弱性スキャン |
| 複雑さ | 低（初期構築が速い） | 高（設定・管理コストが増える） |

---

## チェックリスト

- [ ] Workload Identity Federationを使いサービスアカウントキーを排除している
- [ ] CI用SAの権限は最小限（Artifact Registry書込み + Cloud Runデプロイのみ）
- [ ] テスト・Lintが失敗したらデプロイがブロックされる
- [ ] `main`ブランチへの直接pushを禁止しPRレビューを必須にしている
- [ ] Dockerベースイメージ・GitHub Actionsはバージョン固定している
