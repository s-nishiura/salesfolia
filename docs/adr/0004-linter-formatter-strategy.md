# ADR-0004: LinterおよびFormatterの選定（Prettierの不採用）

## ステータス
採用（Accepted）

## 背景と課題
フロントエンドやテンプレート言語のコードフォーマットにおいて、業界標準として `Prettier` が広く使われている。
しかし、Railsプロジェクト（特にERBテンプレート）においてPrettierを使用すると、設定がブラックボックス化しやすく、意図しないフォーマット崩れや、独自のフォーマット要件に柔軟に対応できないケースが発生しがちである。

## 決定事項
コード品質の均一化において、`Prettier` を不採用とし、各領域に特化した専門のツール群を導入する。

1. **Ruby**: `RuboCop`
   - （`rubocop-shopify`, `rubocop-rails` 等の拡張を使用し、厳格な静的解析を行う）
   - ※ `rubocop-erb` も導入し、ERBファイル内の「Rubyコード部分のみ」を担当させる。
2. **JavaScript / TypeScript / JSON / HTML / CSS**: `Biome`
   - Prettier/ESLintの代替として、高速で設定の手間が少ないBiomeを採用。
3. **ERB (`html.erb`) / HTML (`html`)**: `Herb` エコシステム
   - `@herb-tools/linter`: テンプレートの静的解析。
   - `@herb-tools/formatter`: テンプレートのフォーマット。
   - `@herb-tools/tailwind-class-sorter`: Tailwind CSSのクラス順序の自動整理（formatterのプラグインとして動作）。
   - ※ `html` ファイルは Biome と Herb の両方がカバー範囲となるため、Lefthook等の実行順序は「Herb -> Biome」の順とし、最終的なフォーマットはBiome側の出力を優先する。
4. **YAML**: `yamllint`
5. **GitHub Actions**: `actionlint`

## 結果と影響
### メリット
- ERBテンプレートに対してRailsの文脈に最も適したフォーマット（Herb）が適用される。
- Prettierの過剰な設定やブラックボックスを避け、ツールの責任範囲が明確になる。
- 実行速度が高速化される（Biome等の恩恵）。

### デメリット
- 各ツール（RuboCop, Biome, Herb）を個別にセットアップし、Lefthook等のCIフローに統合する手間がかかる（初期構築時のみ）。

