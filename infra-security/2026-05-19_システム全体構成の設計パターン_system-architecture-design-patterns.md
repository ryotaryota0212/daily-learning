# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「どこに何を置くか」を決める力が、AI時代のエンジニアに最も求められるスキルの一つ。
コードは書けても、リクエストがどこを通ってDBに届くか・どこがボトルネックになるか・何が壊れたら全体が落ちるかを説明できなければ、システムを「成立させる」ことはできない。
この記事では、FastAPI + Neon + Cloud Run + Firebase Auth スタックを前提に、典型的なシステム構成の設計パターンと判断軸を整理する。

---

## 仕組みの要点：典型的な構成レイヤー

```
[Client]
    ↓ HTTPS
[CDN / Edge（Cloudflare）]  ← 静的アセット、WAF、Bot防御
    ↓
[API Gateway / Load Balancer]  ← レート制限、ルーティング
    ↓
[App Server（Cloud Run）]  ← FastAPI、ビジネスロジック
    ↓           ↓           ↓
[Cache(Redis)] [DB(Neon)] [Queue(Pub/Sub)]
                               ↓
                        [Worker(Cloud Run Job)]
```

各レイヤーの責務：

- **CDN/Edge**: 静的ファイル配信、DDoS吸収、地理分散。アプリに到達させないのが基本
- **APIサーバー**: ステートレス設計が必須。Cloud Run はリクエスト単位でスケールするため、サーバー内部に状態を持たせてはいけない
- **キャッシュ**: DBへの読み取り負荷を下げる。読み取り頻度が高く更新頻度が低いデータに使う
- **DB**: トランザクションと整合性が必要なデータのみ。Neon はサーバーレスPGで接続プールに注意
- **キュー**: 非同期・重い処理の分離。APIレスポンスを遅らせない

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| APIサーバーにセッション状態を持つ | Redisなど外部ストアでセッション管理 |
| DBに全データを同期的に書き込む | 重い処理はキューに積んでWorkerが処理 |
| CDNなしで静的ファイルをAPIから返す | CDNにオフロード、APIはAPIだけに集中 |
| 1つのDBに全テーブルを詰め込む | 用途別にDB or スキーマを分離（将来の分割を見越す） |
| Neonにコネクションを張りっぱなし | PgBouncer or Neonの接続プールを使う |

---

## 設計例：ユーザー投稿機能の構成判断

**要件**: ユーザーが投稿 → 画像処理 → 通知送信

```python
# APIエンドポイント（Cloud Run）: 受け付けてすぐキューへ
@app.post("/posts")
async def create_post(data: PostInput, user=Depends(get_current_user)):
    post_id = await db.insert_post(user.uid, data.text)  # DBへ書き込み
    await pubsub.publish("post-created", {"post_id": post_id})  # キューへ
    return {"post_id": post_id}  # すぐレスポンス

# Worker（Cloud Run Job）: キューを消費して重い処理
async def handle_post_created(event):
    post = await db.get_post(event["post_id"])
    await image_service.process(post)   # 画像処理（重い）
    await notification.send(post)       # 通知送信（外部API）
```

**設計のポイント**:
- APIはDBに書いてキューに積むだけ → レスポンスは即時
- 重い処理はWorkerに分離 → APIのスケールと独立してスケール可能
- Workerがクラッシュしてもメッセージはキューに残る → べき等性設計で再処理可

---

## トレードオフ

| 選択 | メリット | コスト・リスク |
|---|---|---|
| キャッシュ追加 | DBへの負荷減・高速化 | キャッシュ無効化の複雑さ、staleデータのリスク |
| キュー導入 | APIのデカップリング、耐障害性 | 結果整合性になる、デバッグが難しい |
| マルチリージョン | 高可用性、低レイテンシ | コスト増、データ同期の複雑さ |
| サービス分割 | 独立デプロイ・スケール | サービス間通信の信頼性管理が必要 |

**判断軸**: 「これがないと壊れるか？」「壊れたとき影響範囲は？」を基準に複雑さを足す。最初はシンプルに保つ。

---

## チェックリスト

- [ ] APIサーバーはステートレスか（Cloud Runの複数インスタンスでも動くか）
- [ ] DBへの接続数は管理されているか（Neonの接続プール設定済みか）
- [ ] 重い処理・外部API呼び出しはキューに分離されているか
- [ ] 読み取り頻度の高いデータにキャッシュ戦略はあるか
- [ ] 各コンポーネントが壊れたとき、何がどう影響するか説明できるか
