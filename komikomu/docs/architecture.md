# アーキテクチャ

## 全体図

```
┌─────────────────────────────┐
│   iOS App (Swift/SwiftUI)   │
│                             │
│  ┌──────────────────────┐   │
│  │ Features (Views/VMs) │   │
│  └──────────┬───────────┘   │
│             │               │
│  ┌──────────▼───────────┐   │
│  │ Repositories         │   │
│  └──────────┬───────────┘   │
│             │               │
│  ┌──────────▼───────────┐   │
│  │ Network (API Client) │   │
│  └──────────┬───────────┘   │
│             │               │
│  ┌──────────▼───────────┐   │
│  │ SwiftData (Cache)    │   │
│  └──────────────────────┘   │
└──────────────┬──────────────┘
               │ HTTPS + JWT
               │
┌──────────────▼──────────────┐
│   Supabase                  │
│  ┌──────────────────────┐   │
│  │ Auth (Apple/Email)   │   │
│  ├──────────────────────┤   │
│  │ Edge Functions (TS)  │   │
│  │  - search-products   │   │
│  │  - get-comparison    │   │
│  │  - affiliate-link    │   │
│  │  - resolve-url       │   │
│  │  - watch-tick (cron) │   │
│  ├──────────────────────┤   │
│  │ Postgres + RLS       │   │
│  │  - watch_items       │   │
│  │  - price_history_d   │   │
│  │  - price_cache       │   │
│  │  - affiliate_clicks  │   │
│  └──────────┬───────────┘   │
└─────────────┼───────────────┘
              │
   ┌──────────┴───────────┐
   ▼          ▼           ▼
 楽天      Yahoo!       Amazon
 API     (バリコマ)     PA-API
```

## モジュール分割（iOS）

```
ios/
├── App/                              # @main・ContentView・AppDelegate
├── Core/
│   ├── Models/                       # SwiftData @Model（1ファイル1モデル）
│   ├── Repositories/                 # 新設：データアクセスの境界層
│   ├── Network/                      # 新設：Supabase クライアント
│   ├── Affiliate/                    # 開示バナー・PR バッジ・必須文言
│   ├── Services/                     # Notification / Entitlement / Browser
│   ├── Extensions/
│   └── Utilities/
│       └── PriceCalculator.swift     # 実質価格算出（送料・ポイント込み）
└── Features/<画面>/{ViewModels,Views}/
```

### 既存2アプリ（コテサク・HomeBuildTracker）との差分

| 観点 | 既存 | コミコム | 理由 |
|---|---|---|---|
| データアクセス | View が `@Query` 直結 | Repository 経由 | ネットワーク経由 EC データのため抽象化必須 |
| Network 層 | なし | `Core/Network/` 新設 | Supabase Edge Functions と通信 |
| 通知 | ローカルのみ | ローカル + APNs（Phase 1.5） | 価格変動通知のため |
| Sign in with Apple | 未使用 | 最初から有効化 | Phase 2 移行時のプロビジョニング再発行を避ける |
| テスト | XCTest / 空 | Swift Testing / 規約モジュールは必ず書く | 規約遵守の自動検証 |
| iOS Deployment Target | 26.5（README は 17+ で不整合） | **17.0 に統一** | SwiftData / Observation 必須 + 配布対象確保 |

## モジュール分割（バックエンド）

```
backend/supabase/
├── config.toml
├── migrations/
│   ├── 0001_init.sql                 # テーブル定義
│   └── 0002_rls.sql                  # Row Level Security 全テーブル
└── functions/
    ├── _shared/
    │   ├── env.ts                    # API キー読込（環境変数）
    │   ├── auth.ts                   # JWT 検証
    │   ├── cors.ts
    │   ├── rate-limiter.ts           # EC API レート制御
    │   └── affiliate.ts              # アフィリエイトリンク生成の正本
    ├── search-products/              # キーワード検索
    ├── resolve-url/                  # 商品 URL → 商品キー解決
    ├── get-comparison/               # 商品キー → 3社最新価格
    ├── affiliate-link/               # 商品キー + EC → アフィリエイト URL
    └── watch-tick/                   # cron: ウォッチ価格更新
```

## データフロー（典型シナリオ）

### シナリオ A：URL を貼って比較

1. ユーザーが楽天/Amazon の URL をペースト
2. iOS → `resolve-url` Edge Function（URL → JAN/型番/正規化キー）
3. iOS → `get-comparison` Edge Function（正規化キー → 3社最新価格）
4. Edge Function は各 EC API を並列に叩く（楽天 API のレート制御をサーバ側で実施）
5. レスポンス内に **アフィリエイト済みリンク** を含めて返却
6. iOS は実質価格を `PriceCalculator` で計算し、最安をハイライト表示
7. ユーザーが商品をタップ → アフィリエイトリンクを **外部 Safari** で開く（Cookie 共有のため `SFSafariViewController` ではなく `UIApplication.shared.open`）

### シナリオ B：お気に入り価格ウォッチ

1. ユーザーが商品を「お気に入り」登録（MVP はローカル SwiftData のみ、Phase 2 でクラウド同期）
2. Supabase の `watch-tick` cron Function が定期実行（1日数回）
3. ウォッチ対象の最新価格を取得、`price_history_daily` に集計値（min/max/avg）を追記
4. 価格下落を検知したら APNs プッシュ通知（Phase 1.5）

## データ層ポリシー

| データ | 保存場所 | TTL/保持方針 |
|---|---|---|
| 商品の最新価格（生レスポンス） | `price_cache` テーブル | `expires_at` 必須、cron で物理削除（24h以内） |
| 価格履歴 | `price_history_daily` テーブル | 日次集計値のみ、生データは保存しない |
| ウォッチリスト | `watch_items` テーブル + iOS SwiftData | ユーザーごと、無制限 |
| お気に入り（MVP） | iOS SwiftData | Phase 2 でクラウド同期 |
| ユーザー認証 | Supabase Auth | Apple JWT / Email |
| アフィリエイト ID | サーバ環境変数のみ | クライアントに露出させない |

詳細は [ADR 0004: price-cache-ttl](decisions/0004-price-cache-ttl.md) 参照。

## セキュリティ

- 各テーブルに RLS（Row Level Security）：`user_id = auth.uid()` 制約
- Edge Functions は Supabase Auth の JWT を検証してから処理
- アフィリエイト ID / EC API キーは **環境変数のみ**、コードにハードコードしない
- iOS から EC API を直接叩かない（API キー露出回避 + サーバ側レート制御）

## 拡張性

- Web / Android / Chrome 拡張は同じ Edge Functions を叩く設計
- `shared/openapi.yaml` を正本に、各クライアントは型を生成して使用
- EC ソースの追加（au PAY、Qoo10 等）は `_shared/` に Adapter を1つ足すだけ
