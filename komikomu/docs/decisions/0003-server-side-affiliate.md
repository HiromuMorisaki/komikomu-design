# ADR 0003: アフィリエイトリンク生成はサーバ側で行う

## Status

Accepted (2026-05-30)

## Context

各 EC モールのアフィリエイトリンクには、トラッキング ID（アソシエイト ID、楽天アフィリエイト ID、バリコマ SID 等）を URL パラメータで付与する必要がある。

このトラッキング ID の管理場所として、以下が考えられる：

1. iOS / Web / Chrome 拡張のクライアントコードに埋め込む
2. サーバ（Supabase Edge Function）の環境変数で管理し、リンク生成も サーバ側で行う

クライアント側に埋め込んだ場合、**バイナリ解析・拡張機能のソース閲覧で ID が露出する** リスクがある。

## Decision

**アフィリエイトリンクの生成は、すべて Supabase Edge Functions（`affiliate-link` および各 EC API 結果に埋め込む処理）で行う**。

- ID は Supabase の環境変数（Secrets）で管理し、コードベースには登場しない
- iOS / Web / Chrome 拡張は、サーバから返された URL をそのまま開くだけ
- リンク生成ロジックは `backend/supabase/functions/_shared/affiliate.ts` に集約

## Alternatives Considered

### クライアント側でリンク生成

- **Pro**：実装が単純（リクエスト 1 つ減る）
- **Pro**：レイテンシが低い
- **Con**：**ID が露出する**（iOS バイナリの strings 解析、Chrome 拡張の `chrome-extension://` URL から JS が読める）
- **Con**：ID 変更時にアプリ更新が必要、即時対応不可
- **Con**：将来の iOS / Web / 拡張間で重複コードが発生

### iOS のみ Edge Function 経由、Web / 拡張はクライアント側

- **Con**：一貫性がなく規約面・実装面で複雑化
- **Con**：拡張機能は最もコード閲覧が容易なため、ここで露出するのが最悪

## Consequences

### Positive

- アフィリエイト ID が露出しない
- ID 変更がサーバ側のみで完結（クライアント更新不要）
- iOS / Web / Chrome 拡張で同じリンクが生成されることが保証される（規約遵守が一貫）
- ASP 規約変更時の影響範囲が `_shared/affiliate.ts` に集中、メンテが容易

### Negative / Trade-offs

- リンクを開くたびに 1 ホップ多い（Edge Function 呼び出し）
- ただし結果は **API レスポンスにあらかじめ埋め込んで返す** ことで、商品表示時に追加コール不要にできる
- Edge Function 障害時はリンク生成が止まる → キャッシュ済みリンクは利用可能、新規生成のみ影響

### 実装ルール

- `_shared/affiliate.ts` が **アフィリエイトリンク生成の唯一の正本**
- どの Edge Function も、リンクを返す時は必ずこの関数を経由する
- 環境変数の命名規則：`AFFILIATE_RAKUTEN_ID`、`AFFILIATE_AMAZON_TAG`、`AFFILIATE_VALUECOMMERCE_SID`
- 環境変数未設定時は **リンクを返さずエラーにする**（フォールバック禁止 = 規約違反を未然に防ぐ）
- 必須テスト：[affiliate-policy.md §7](../affiliate-policy.md#7-設計実装ガイドライン) 参照
