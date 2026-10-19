# ADR 0001: Supabase をバックエンドの主軸として採用する

## Status

Accepted (2026-05-30)

## Context

コミコムは iOS フロント + バックエンド API + 将来 Web / Chrome 拡張の構成で、個人開発として 1 人で開発・運用していく必要がある。

要件：

- 認証（Apple Sign In、Email、将来 Google）
- データベース（ユーザー、ウォッチリスト、価格集計値、キャッシュ）
- サーバサイド処理（EC API 集約、アフィリエイトリンク生成、レート制御）
- 定期実行（価格ウォッチの定期取得）
- iOS / Web / Chrome 拡張の共通バックエンド

候補：

1. Supabase（Auth + Postgres + Edge Functions on Deno）
2. Node.js (TypeScript) + Hono on Cloudflare Workers + D1 + Workers KV
3. Python (FastAPI) on Cloud Run + Cloud SQL
4. Firebase（Auth + Firestore + Cloud Functions）

## Decision

**Supabase をバックエンドの主軸とする**。

- Auth：Supabase Auth（Apple / Email / 後に Google）
- DB：Supabase Postgres + RLS（Row Level Security）
- サーバ処理：Supabase Edge Functions（Deno + TypeScript）
- 定期実行：Supabase Scheduled Functions

将来、パフォーマンス・コスト・特殊な要件で必要が出れば、**特定機能を Cloudflare Workers にオフロード** する余地を残す（例：高頻度のアフィリエイトリンクリダイレクト）。

リージョンは `ap-northeast-1`（Tokyo）を指定する。

## Alternatives Considered

### Cloudflare Workers + Hono + D1 単独案

- **Pro**：コスト・レイテンシ・型安全性（TypeScript）が魅力的
- **Con**：認証を一から設計する必要あり（WorkOS や Auth0 を別途導入することになる）
- **Con**：1 人運用で Auth、DB、ストレージ、リアルタイム、管理画面を別々に組むコストが大きい
- **Con**：Auth + DB + 管理画面 + Storage が一発で揃う Supabase の総合力に劣る

### Python (FastAPI) on Cloud Run

- **Con**：冷起動・課金・IAM 管理が 1 人運用には重い
- **Con**：TypeScript で Web / Chrome 拡張とスキーマ共有する将来構想と噛み合わない

### Firebase

- **Con**：Firestore の柔軟な構造は魅力だが、価格履歴の集計値や RLS 相当の制御が Postgres + RLS より煩雑
- **Con**：Apple Sign In + JWT 検証フローが Supabase より工数増

## Consequences

### Positive

- 1 人運用で Auth + DB + サーバ処理 + Storage を一気に揃えられる
- Postgres + RLS で型安全 & 厳密な権限制御
- 管理画面（Supabase Studio）でデータが目視できる
- Edge Functions が TypeScript なので、将来 Web / Chrome 拡張とスキーマ共有が容易
- 無料枠が個人開発の MVP に十分

### Negative / Trade-offs

- Deno ベースの Edge Functions エコシステムは Node.js より狭い（一部 npm パッケージが使えない）
- Supabase 自体の障害が起きると全機能停止リスク（マネージドサービスの宿命）
- スケール後の費用が Cloudflare Workers より高めになる可能性

### 移行余地

- 仮に Supabase を捨てる必要が出ても、Postgres + 標準的な Auth プロトコルなので、コア資産は移植可能
- Edge Functions を Hono + Workers に書き換える設計余地を、`_shared/` の構造で確保しておく
