# コミコム

> 送料・ポイント込みで実質最安を比較するショッピングアプリ

## 概要

コミコムは、Amazon・楽天市場・Yahoo!ショッピングを横断して、送料・ポイント還元を加味した「実質価格」で最安を提示する iOS アプリです。

「ちゃんと比べたら安いのに、面倒だから1サイトで買ってしまう」状態を、最小操作で解消します。

**キャッチコピー**：送料も、ポイントも、ぜんぶコミで。

**名前の由来**：「コミ」（送料・ポイント "込み"）+「コム」（動詞 "込む" / 副次的に .com）

## ターゲットユーザー

- 日常的に EC サイトを使う 20〜50 代
- 楽天経済圏 / Amazon プライム / PayPay 経済圏のいずれかに属している
- 比較する手間を惜しんで、結局1サイトで買ってしまう層

## 開発フェーズ

| Phase | 対象 | 内容 |
|---|---|---|
| 1 (MVP) | iOS | 楽天 API による価格取得・実質価格比較・ローカルお気に入り・StoreKit 2 土台 |
| 1.5 | iOS | 価格変動の APNs プッシュ通知 |
| 2 | クラウド | アカウント連携・お気に入り/履歴同期・有料プラン本格化 |
| 3 | Web / Android | 同一バックエンドでクロスプラットフォーム展開 |
| 4 | Chrome 拡張 | Amazon 商品ページ → 楽天/Yahoo 同等品サイドバー表示 |

## 技術スタック

| レイヤ | 採用 |
|---|---|
| iOS | Swift / SwiftUI / SwiftData |
| iOS Deployment Target | 17.0 |
| iOS 外部依存 | Supabase Swift SDK のみ（SPM 経由） |
| バックエンド | Supabase（Auth + Postgres + RLS + Edge Functions on Deno） |
| 認証 | Apple Sign In / Email / Google (Phase 2) |
| 課金 | StoreKit 2 |
| テスト | Swift Testing |
| 将来 Web / Chrome 拡張 | React + TypeScript（同じ Edge Functions を共有） |

## ブランディング

| 項目 | 値 |
|---|---|
| App 名 | コミコム |
| サブタイトル | 送料・ポイント込みで実質最安を比較 |
| Bundle Name | コミコム-実質最安価格比較 |
| Bundle ID | com.h-morisaki.Komikomu |
| App Group ID | group.com.h-morisaki.Komikomu |

## リポジトリ構成（予定）

```
.
├── README.md                # 本ファイル
├── ios/                     # iOS アプリ（Swift/SwiftUI）
├── backend/                 # Supabase（マイグレーション・Edge Functions）
├── shared/                  # iOS/Web/拡張機能で共有する型定義（OpenAPI 等）
└── docs/                    # 設計ドキュメント
    ├── architecture.md
    ├── affiliate-policy.md  # ASP 規約サマリ（最重要・半年ごと見直し）
    ├── api-strategy.md
    └── decisions/           # Architecture Decision Records (ADR)
        ├── 0001-supabase-backend.md
        ├── 0002-no-scraping.md
        ├── 0003-server-side-affiliate.md
        └── 0004-price-cache-ttl.md
```

## 主要ドキュメント

- [アーキテクチャ](docs/architecture.md)
- [アフィリエイト規約サマリ](docs/affiliate-policy.md)
- [EC API 連携戦略](docs/api-strategy.md)
- [Architecture Decision Records](docs/decisions/)

## 厳守ポリシー

1. **スクレイピング禁止**：公式 API のみ使用（[ADR 0002](docs/decisions/0002-no-scraping.md)）
2. **アフィリエイトリンクはサーバ側で生成**：ID をクライアントに露出させない（[ADR 0003](docs/decisions/0003-server-side-affiliate.md)）
3. **価格情報は集計値のみ永続化**：生データは TTL 付きキャッシュ（[ADR 0004](docs/decisions/0004-price-cache-ttl.md)）
4. **比較ロジックは中立**：報酬率による並べ替えは行わない
5. **景表法・ステマ規制対応**：PR バッジ + 開示バナーで明示

## 開発状況

現在 **Step 1（設計フェーズ）完了**。本ドキュメント群が初期成果物。
次は Step A（iOS / backend スケルトン）、Step B（楽天 API 連携 Edge Function 実装）に進む予定。
