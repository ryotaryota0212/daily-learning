# ハッシュテーブルの仕組み（衝突解決・リサイズ・計算量）

## 概要

ハッシュテーブルはキーと値のペアをO(1)で検索・挿入・削除できるデータ構造。
PythonのdictやJavaのHashMapがこれに相当し、キャッシュ・重複チェック・集計など実務で最頻出。
「なぜO(1)なのか」と「どんなときにO(n)に劣化するか」を理解することが重要。

## 仕組みの要点

### 基本構造

- **ハッシュ関数**: キー → 整数インデックス に変換する関数
- **バケット配列**: ハッシュ値をインデックスとして使う固定長配列
- 流れ: `key` → `hash(key) % capacity` → バケット番号 → 値を格納

```
key="alice"  →  hash("alice")=12345  →  12345 % 8 = 1  →  bucket[1]
key="bob"    →  hash("bob")=67890    →  67890 % 8 = 2  →  bucket[2]
```

### 衝突（Collision）

異なるキーが同じバケットに割り当てられる現象。必ず起こりうる。

**解決方法1: チェイン法（Separate Chaining）**
- 各バケットにリンクドリストを持つ
- 衝突時はリストに追記
- 最悪O(n)だが、負荷率が低ければほぼO(1)

**解決方法2: オープンアドレス法（Open Addressing）**
- 衝突したら別のバケットを探す（線形探索・二次探索・ダブルハッシュ）
- キャッシュ効率が高い（配列連続アクセス）
- 負荷率が高くなるほど極端に劣化

### 負荷率（Load Factor）

```
負荷率 = 格納要素数 / バケット総数
```

- Python dict: 負荷率が2/3を超えたら拡張
- 一般的な閾値: 0.7〜0.75
- 負荷率が高い → 衝突増加 → 検索O(n)に近づく

### リサイズ（動的拡張）

1. 容量を約2倍にした新しい配列を確保
2. 全要素を**再ハッシュ**して新配列に移し替え（O(n)コスト）
3. 旧配列を破棄

→ リサイズは高コストだが、頻度が低いため**償却O(1)**が成立する。

## 計算量・パフォーマンス特性

| 操作 | 平均 | 最悪（衝突多発） |
|------|------|-----------------|
| 検索 | O(1) | O(n) |
| 挿入 | O(1) | O(n) |
| 削除 | O(1) | O(n) |
| リサイズ | — | O(n) |

- 最悪ケースはすべてのキーが同一バケットに集中した場合
- 良いハッシュ関数 + 適切な負荷率管理で実質O(1)

## コード例（Python）

```python
# シンプルなチェイン法ハッシュテーブル
class HashTable:
    def __init__(self, capacity=8):
        self.capacity = capacity
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]

    def _index(self, key):
        return hash(key) % self.capacity

    def set(self, key, value):
        idx = self._index(key)
        for i, (k, v) in enumerate(self.buckets[idx]):
            if k == key:
                self.buckets[idx][i] = (key, value)
                return
        self.buckets[idx].append((key, value))
        self.size += 1
        if self.size / self.capacity > 0.7:
            self._resize()

    def get(self, key):
        for k, v in self.buckets[self._index(key)]:
            if k == key:
                return v
        raise KeyError(key)

    def _resize(self):
        old = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0
        for bucket in old:
            for k, v in bucket:
                self.set(k, v)
```

## よくある誤解・落とし穴

- **「常にO(1)」は誤り**: 最悪ケースはO(n)。悪意あるキー入力でDoSになる（Hash Flooding攻撃）
  - Python 3.3以降はランダムシード（`PYTHONHASHSEED`）でこれを緩和
- **順序の保証**: Python 3.7以降のdictは挿入順を保持するが、ハッシュテーブル一般では保証されない
- **ミュータブルなオブジェクトはキーにできない**: ハッシュ値が変わると取得不能になるため（Pythonでlistはキー不可）
- **負荷率を下げればいいわけではない**: メモリ消費が増大するトレードオフがある

## 実務での使いどころ

**FastAPI + Neon + Cloud Runスタックとの関連**

- **FastAPIのルーティング**: エンドポイントのマッピングに内部でdict（ハッシュテーブル）を使用
- **Neon（PostgreSQL）**: `=`や`IN`の条件でHash Join・Hash Aggregateが使われる。`EXPLAIN ANALYZE`で確認可能
- **APIレスポンスのキャッシュ**: Redisのデータ構造はハッシュテーブルがベース。`HSET/HGET`で部分更新可能
- **重複排除・集計**: ログ処理や集計バッチで `dict` や `Counter` を使うと線形時間で処理できる
- **セッション管理**: セッションIDをキー、ユーザー情報を値とするハッシュテーブルがセッションストアの基本
