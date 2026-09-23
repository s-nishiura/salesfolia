# フェーズ0: 開発環境構築 (Environment)

本プロジェクトにおけるローカル開発環境は、チーム全体でバージョン差異を無くし、クリーンな環境を保つために `Devbox` と `Mise` を組み合わせたモダンなツールチェーンを採用する。

## 1. ツールチェーンとバージョン管理の責任分界点
プロジェクトの再現性を担保するため、各ツールの役割を以下のように厳密に定義する。

- **Devbox (`devbox.json`)**
  - `PostgreSQL 18` などのDBおよびシステム依存パッケージを管理する。
  - Git/CI支援ツール（`lefthook`, `act`, `actionlint`, `pinact`, `gh` 等）もここで管理する。
  - `mise` 自体のインストールも Devbox 経由で行う。
- **Mise (`mise.toml`)**
  - `Ruby`, `Bun`, `Node.js` などの言語ランタイムのバージョンを指定・固定する。
  - 基本的なJSランタイム・パッケージ管理は `Bun` を使用し、`Node.js` は開発環境におけるフォールバック用（Playwright等のBunだけでエラーになるツール用）としてのみ使用する。

※ アプリケーションレベルのライブラリ（Rails本体など）については、それぞれの言語標準のパッケージマネージャ（`Gemfile` や `package.json`）でバージョン指定を行う。

## 2. 技術スタック・アーキテクチャ概要
本プロジェクトで採用する主要な技術スタックを以下に定義する。

- **バックエンド**: Ruby on Rails
- **フロントエンド**: Hotwire (Turbo + Stimulus)
  - SPA等の重厚なフロントエンド技術は避け、Railsの標準であるサーバーサイドレンダリング(SSR)を活かす。
  - 専用業務機ライクな「高度なキーボード制御」は Stimulus を用いて局所的に実装する。
- **UIコンポーネント**: `ViewComponent` + `DaisyUI` (Tailwind CSS v4ベースのUIライブラリ)
  - ※ `reactionview` はデバッグ用に出力される `span` タグによってHTMLの階層構造が変わってしまい、親子関係のDOM構造に依存したDaisyUIのスタイルが正しく当たらなくなる相性問題があるため不採用とした。
- **データベース**: `PostgreSQL 18`（マルチテナントデータおよびJSONBカラムを活用）
  - `citext` 拡張を有効化し、メールアドレス等の大文字小文字を区別しない比較を実現する。
- **権限・アクセス制御**: 階層的な権限管理システム。
  - **担当グループ（ロール）権限**: ユーザーが所属するグループベースでの権限付与。
  - **ユーザー個別権限**: ユーザー単体に対しても権限のオーバーライドが可能。
  - **適用優先度**: グループ設定 ＜ 個人設定

## 3. ローカル開発URL方針
- **サブドメイン方式**を採用し、テナントごとに独立したURL（例: `tenant.salesfolia.dev`）でアクセスする設計とする。
- ローカル開発環境では、設定不要でワイルドカードサブドメインを `127.0.0.1` に解決する **`lvh.me`** を使用する（例: `http://[slug].lvh.me:3000`）。
  - Windows + WSL2 環境での `hosts` ファイル書き換えやワイルドカードDNS設定は不要。
  - オフライン環境での動作は保証しない（外部DNSに依存）。

## 4. 環境構築手順 (概要)
1. `devbox init` にて `devbox.json` を生成。
2. Devbox経由でPostgreSQLをインストール・サービス起動。
3. `mise use` (または `mise.toml`) で Ruby, Bun, Node.js のバージョンを固定。
4. `rails new . -d postgresql -c tailwind --javascript=bun` などのコマンドでRailsプロジェクトを初期化する。

## 4.1. 国際化 (i18n) とタイムゾーン
- **デフォルトロケール**: `ja`（日本語）のみを初期対象とする。ただし将来的に他言語を追加できるよう、ハードコードせず `config.i18n.available_locales` で管理する。
- **i18nライブラリ**: `rails-i18n`（Rails標準メッセージの日本語化）および `rodauth-i18n`（Rodauth認証画面の日本語化）を導入する。
- **タイムゾーン**: アプリケーション層のデフォルトは `Asia/Tokyo` とするが、データベースには **UTC** で保存する。将来的な他タイムゾーン対応に備え、表示時にのみユーザーのタイムゾーンで変換する設計とする。

## 4.2. Railsデフォルト構成
以下はRails標準（`rails new` で生成）のまま使用し、特別なカスタマイズは行わない。
- **アセットパイプライン**: `Propshaft`（Rails 8標準）
- **バックグラウンドジョブ / キャッシュ / WebSocket**: `Solid Queue` / `Solid Cache` / `Solid Cable`（Rails 8標準のDB-backed構成）
- **デプロイ**: `Kamal`（Rails 8標準のコンテナデプロイツール）

## 5. セキュリティ
- **パッケージレジストリ**: サプライチェーン攻撃対策として、Gemおよびnpmのソースに Takumi Guard (Flatt Security) を使用する。
  - Gemソース: `https://rubygems.flatt.tech`
  - npmソース (Bun): `https://npm.flatt.tech`
- **静的解析 (Security)**: 以下の脆弱性スキャン・監査ツールを導入し、コードや依存関係の安全性を担保する。
  - `brakeman`: Railsコードの脆弱性スキャン。
  - `bundler-audit`: Gem依存関係の既知CVEチェック。
  - `bun audit`: npmパッケージ依存関係の既知CVEチェック。

## 6. テスト・品質保証 (QA & Testing)
エンタープライズ向けのSaaS（会計・販売管理）において品質は最重要であるため、以下のテスト基盤を導入する。

- **ユニット / リクエストテスト**: `RSpec` を採用。
  - プロジェクト固有のビジネスロジックや権限周りのテストを網羅的に記述する。
  - `simplecov` を導入し、テストカバレッジを可視化・品質の指標として管理する。
  - `shoulda-matchers` を導入し、モデルのバリデーションやアソシエーションのテストを簡潔に記述する。
- **テストデータ生成**: `FactoryBot` ＋ `Faker` を採用。
  - 全てのデータが `tenant_id` に依存するため、Factoryの作成時は必ずテナント情報が紐づくように設計する。
- **E2E (System) テスト**: `Playwright` を採用。
  - `playwright-ruby-client` + `capybara-playwright-driver` を介してRSpecから実行する。
  - 「キーボード操作のみで完結する」というコアUXを保証するため、複雑なキーボード操作やフォーカス移動のテストをPlaywright経由で自動化する。
- **パフォーマンス**: `prosopite` を導入し、N+1クエリを開発・テスト環境で自動検出する。

## 7. Linter / Formatter / コード品質
コード品質を均一化するため、以下のツールを導入する。

- **RuboCop**: Rubyコードの静的解析・フォーマット。以下の拡張を使用する。
  - `rubocop-erb`: ERBファイル内の「Rubyコード部分のみ」を静的解析する。
  - `rubocop-shopify`: 大規模商用環境のベストプラクティスをベースとした厳格なルールセット。
  - `rubocop-rails`, `rubocop-rspec`, `rubocop-rspec_rails`: Rails/RSpec固有のルール。
  - `rubocop-factory_bot`, `rubocop-capybara`, `rubocop-performance`: 各用途向けルール。
- **Biome**: JavaScript, TypeScript, JSON, HTML, CSS の Linter + Formatter として採用（Prettierは導入しない）。
- **Yamllint**: YAMLファイルの構文チェック・フォーマット（Devboxで管理）。
- **Actionlint**: GitHub Actionsのワークフロー定義ファイルの静的解析（Devboxで管理）。
- **Herb**: `html.erb` および `html` テンプレートのフォーマット・解析として採用（以下のパッケージ群を使用）。
  - `@herb-tools/linter`: 静的解析。
  - `@herb-tools/formatter`: フォーマット。
    - `@herb-tools/tailwind-class-sorter`: formatterのプラグインとして導入し、Tailwind CSSのクラス順序を自動整理する。
  - ※ `html` ファイルは Biome と Herb の両方が担当するため、フォーマット時は「Herb を実行した後に Biome を実行（Biomeのフォーマットを優先）」する順番とする。

## 8. Git・コミット品質管理
- **Lefthook**: Git Hookの管理ツール（`lefthook.yml` で定義）。コミット前のLint/Formatチェックを自動化する。
- **Commitlint**: コミットメッセージのConventional Commits規約準拠チェック（`commitlint.config.js` で定義）。
- **i18n-tasks**: 未使用・未翻訳のi18nキーを静的解析で検出する。
