# ADR 0004: 価格データは TTL 付きキャッシュ + 集計値のみ永続保存

## Status

Accepted (2026-05-30)

## Context

EC API から取得した価格データの保存方針を決める必要がある。

要件：

- 同一ユーザー / 別ユーザーの重複リクエストを減らし、API レート制限を超過しない
- 価格履歴グラフ（Phase 2 機能）を実現する
- 各 ASP の規約「価格情報の取り扱い」を遵守する
  - 楽天 API：取得から 24 時間以内の情報を使う運用が推奨される
  - Amazon PA-API：価格情報の保存期間に制約あり（運営規約参照）

候補：

1. 生レスポンスを永続保存（履歴も全部生データから集計）
2. TTL 付きで生データを一時保存 + 日次集計値のみ永続保存
3. キャッシュなし、毎回 API を叩く

## Decision

**選択肢 2 を採用する**：

- 生レスポンスは `price_cache` テーブルに **`expires_at` 必須カラム付き** で保存。デフォルト TTL は **24 時間**
- cron で `expires_at < NOW()` のレコードを **物理削除** する
- 価格履歴は `price_history_daily` テーブルに **日次集計値のみ**（min / max / avg / count）を永続保存
- 生データは履歴目的では一切保存しない

## Alternatives Considered

### 生レスポンスを永続保存

- **Pro**：履歴グラフの粒度を後から自由に変えられる
- **Con**：**各 ASP の価格情報の取り扱い規約に抵触するリスク**
- **Con**：DB サイズが線形に膨らみ、コスト・パフォーマンス劣化

### キャッシュなし

- **Pro**：実装が単純、規約面で最も安全
- **Con**：楽天 API の 1 秒 1 リクエスト制限に容易に到達
- **Con**：同じ商品を別ユーザーが見るたびに API コール、無駄が多い

## Consequences

### Positive

- 規約遵守が DB スキーマレベルで強制される
- API レート制限超過のリスクを大幅に下げられる
- 履歴グラフは集計値で実現可能（min/max/avg があれば十分）
- DB サイズが日次集計の線形でしか伸びない

### Negative / Trade-offs

- 履歴グラフの粒度は「日次」が下限になる（時間単位の変動は見えない）
- ただし「実質最安を提示する」コアバリューには時間単位の粒度は不要
- TTL 切れデータの物理削除を cron で確実に実行する必要あり（cron が止まると DB 肥大化）

### 実装ルール

`price_cache` テーブル定義：

```sql
CREATE TABLE price_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_key TEXT NOT NULL,
  ec_source TEXT NOT NULL,
  raw_response JSONB NOT NULL,
  fetched_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,  -- 必須
  UNIQUE (product_key, ec_source)
);
CREATE INDEX idx_price_cache_expires ON price_cache(expires_at);
```

`price_history_daily` テーブル定義：

```sql
CREATE TABLE price_history_daily (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_key TEXT NOT NULL,
  ec_source TEXT NOT NULL,
  date DATE NOT NULL,
  min_price_jpy INTEGER NOT NULL,
  max_price_jpy INTEGER NOT NULL,
  avg_price_jpy INTEGER NOT NULL,
  sample_count INTEGER NOT NULL,
  UNIQUE (product_key, ec_source, date)
);
```

運用ルール：

- 削除 cron：1 日 1 回 `DELETE FROM price_cache WHERE expires_at < NOW();`
- iOS 側 `PriceQuote` モデルも `expiresAt: Date` を必須カラムにする
- `expires_at` を過ぎたデータは UI に表示しない（リクエスト時に取り直す）
