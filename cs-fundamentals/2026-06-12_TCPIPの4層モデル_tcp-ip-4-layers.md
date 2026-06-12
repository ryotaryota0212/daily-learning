# TCP/IPの4層モデルと各層の役割

## 概要

TCP/IPモデルはインターネット通信の基盤となる4層のプロトコルスタック。OSI 7層モデルの実用版として設計され、現代のすべてのネットワーク通信に使われている。

FastAPIのリクエスト処理・Cloud RunのHTTPS終端・NeonのDB接続はすべてこの層の上で動いており、「なぜ接続が切れるか」「なぜ遅延が発生するか」を診断するための必須知識。

---

## 仕組みの要点

### 4層の構成

```
┌─────────────────────────────────┐
│  4. アプリケーション層            │  HTTP, HTTPS, DNS, SMTP
│     (何を送るか)                 │
├─────────────────────────────────┤
│  3. トランスポート層              │  TCP, UDP
│     (どう確実に届けるか)          │
├─────────────────────────────────┤
│  2. インターネット層              │  IP, ICMP
│     (どこへ届けるか)             │
├─────────────────────────────────┤
│  1. ネットワークアクセス層         │  Ethernet, Wi-Fi
│     (物理的にどう送るか)          │
└─────────────────────────────────┘
```

### 各層の詳細

**1. ネットワークアクセス層（リンク層）**
- 同じLAN内のマシン間でフレームを送受信
- MACアドレスで識別（物理デバイスに固定）
- EthernetフレームにIPパケットをカプセル化

**2. インターネット層**
- IPアドレスで宛先を特定し、ルーター間をホップして届ける
- **ベストエフォート**：到達保証なし、順序保証なし
- IPパケットの最大サイズ（MTU）は通常1500バイト。超える場合はフラグメント化

**3. トランスポート層**
- **TCP**：3ウェイハンドシェイク → 順序保証・再送制御・輻輳制御。信頼性重視
- **UDP**：ヘッダー最小限、送りっぱなし。低遅延重視（動画ストリーミング、DNS）
- ポート番号でプロセスを識別（HTTP=80, HTTPS=443）

**4. アプリケーション層**
- アプリが実際に使うプロトコル
- HTTP/HTTPS, WebSocket, DNS, SSH, SMTP など
- TLSはここで動作（TCP層の上で暗号化）

### データの流れ（カプセル化）

送信側は上から下へ各層のヘッダーを付加：
```
HTTPリクエスト
  → [TCPヘッダー + HTTPデータ] (セグメント)
  → [IPヘッダー + セグメント]  (パケット)
  → [MACヘッダー + パケット]  (フレーム)
  → 物理信号として送出
```
受信側は逆順にヘッダーを剥がして処理する。

---

## パフォーマンス特性

| プロトコル | レイテンシ | 信頼性 | 主な用途 |
|-----------|-----------|-------|---------|
| TCP       | +RTT分高い | 高い  | HTTP, DB接続 |
| UDP       | 最小      | 低い  | DNS, 動画, QUIC |

**TCPの3ウェイハンドシェイク**：接続確立だけで1RTT消費。
- Client → SYN → Server
- Server → SYN-ACK → Client
- Client → ACK → Server
- その後データ送受信開始

HTTPS(TLS 1.3)では合計2RTT（TCP 1RTT + TLS 1RTT）が最小コスト。

---

## コード例

```python
import socket

# TCPの基本：3ウェイハンドシェイク後にデータ送受信
def raw_http_request(host: str, port: int = 80) -> str:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((host, port))          # SYN/SYN-ACK/ACK
        request = f"GET / HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n"
        s.sendall(request.encode())
        response = b""
        while chunk := s.recv(4096):    # TCPが順序保証して届ける
            response += chunk
    return response.decode(errors="replace")

# UDPの基本：送りっぱなし（DNS問い合わせなどに使われる）
def udp_send(host: str, port: int, data: bytes) -> None:
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.sendto(data, (host, port))    # ハンドシェイクなし、即送信
```

---

## よくある誤解・落とし穴

- **「TCPは絶対に届く」は誤り**：OSがリトライを繰り返すが、タイムアウト後は接続断になる。アプリ側の再試行ロジックは必要。
- **IPアドレスとMACアドレスの混同**：IPはルーター越えで変わらないが、MACはホップごとに書き換わる（ルーターが書き換える）。
- **ポートはプロセスの識別子**：同一IPでも複数サービスが動けるのはポートで区別するから。Cloud Runの`:8080`がその例。
- **TLSはアプリケーション層**：TCPの信頼性の上に乗っている。TCPが届けた後、TLSが復号する。
- **UDP = 低レベル = 危険ではない**：HTTPSと同じくUDPの上にQUICが動き、HTTP/3ではUDP+QUICが信頼性を担保する。

---

## 実務での使いどころ

**FastAPI (Cloud Run)**
- Cloud RunはHTTPS（TCP 443）を終端してコンテナに転送。コンテナは`:8080`でListenするだけでよい。
- `uvicorn`がTCPソケットを管理。`keepalive_timeout`設定でTCP接続の再利用コストを下げられる。

**Neon (PostgreSQL)**
- PostgreSQLはTCPベース。接続コストはTCP 1RTT + TLS 1RTT + PostgreSQL認証。
- コネクションプーリング（PgBouncer/Neon Proxy）はこのハンドシェイクコストを減らすために存在する。

**デバッグ時の確認コマンド**
```bash
# TCP接続の状態確認
ss -tn state established

# パケット経路の確認（IPレイヤー）
traceroute 8.8.8.8

# DNS（UDPポート53）の確認
dig @8.8.8.8 example.com
```

**タイムアウト設計の基本**：TCPの再送タイムアウトはOSが管理（デフォルト75秒〜）。FastAPIのhttpxクライアントでは`connect=5, read=30`秒程度を明示設定するのが実務の定石。
