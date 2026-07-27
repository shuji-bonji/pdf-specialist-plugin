# pdf-specialist-plugin

[English](./README.md)

PDF 専門家サブエージェント **pdf-specialist** の統合プラグイン。
1 プラグインで「エージェント定義 + PDF Family 4 MCP サーバ + pdf-trust / pdf-publish Skill」が揃う。

> 設計書: PDF 専門家エージェント設計書 パターン1（Claude サブエージェント構成）の実装。
> **現状 v0.2.0 も未実運用**（v0.1.0 が初版、v0.2.0 で pdf-trust 0.3.0 を再同期し版数要件を追記）。
> 実行利用の結果で詳細（tools パターン・model・発火条件）を補正する。

## 構成

```
pdf-specialist-plugin/
├── .claude-plugin/plugin.json   # マニフェスト + mcpServers（PDF Family 4 サーバを npx 接続）
├── agents/pdf-specialist.md     # サブエージェント定義（委譲トリガー + 絶対規則）
└── skills/
    ├── pdf-trust/               # 受入監査 Skill（同梱コピー）
    └── pdf-publish/             # 納品パイプライン Skill（同梱コピー）
```

## インストール

```
/plugin marketplace add shuji-bonji/claude-plugins
/plugin install pdf-specialist
```

### 必要な環境設定

| 変数 | 対象 | 必須 | 内容 |
|---|---|---|---|
| `PDF_SPEC_DIR` | pdf-spec | pdf-spec を使うなら必須 | 仕様 PDF コーパスのディレクトリ。未設定だと pdf-spec は起動失敗するが、他 3 サーバと Skill は動作する |
| `PDF_VERIFY_VERAPDF` | pdf-verify | 任意 | veraPDF 実行パス。無ければ PATH 探索 → 内蔵ルールに縮退 |
| `PDF_VERIFY_TRUST_ANCHORS` | pdf-verify | 任意 | 信頼アンカー証明書ディレクトリ。無いと verdict は use_with_caution 止まり |
| `PDF_WRITER_FONT` | pdf-writer | 日本語出力に実質必須 | 単一フェイス .ttf/.otf（Noto Sans JP static 版推奨） |

環境変数はシェル環境（launchd / .zshenv 等）に設定する。プラグインの mcpServers は起動時に環境を継承する。

### バージョン要件

- pdf-verify-mcp **v0.7.0+**（evaluate_policy。pdf-trust の 4 値判定に必須）／
  **v0.10.0+ 推奨**（`verify_integrity` のリビジョン間オブジェクト単位差分。
  「署名後に何が変わったか」をオブジェクト単位で言えるようになる。4 値判定自体は不変）
- pdf-reader-mcp **v0.10.0+** 推奨（`locate_objects` = オブジェクト番号 → ページ + 矩形。
  上の差分を位置に落とすのに要る）
- pdf-writer-mcp **v0.15.0+** 推奨（preserveSignatures / tag_form_fields）

`.claude-plugin/plugin.json` は `@latest` で接続するため通常は満たされる。

## 使い方

メインエージェントに普通に依頼すれば、description の発火条件により pdf-specialist へ委譲される:

- 「この契約書 PDF、信用していい？」→ pdf-trust（profile=contract）
- 「この Markdown を PDF/UA 準拠のタグ付き PDF にして」→ pdf-publish
- 「タグ付き PDF で文書タイトルは必須？根拠条文は」→ pdf-spec 照会

## 設計原則（要点）

1. **verdict の上書き禁止** — 判定は evaluate_policy / veraPDF。エージェントは解説のみ
2. **宣言と適合の分離** — ensure_* を呼んだら対応 flavour の validate_conformance を必ず通す
3. **絶対パス + outputPath 必須** — base64 溢れ（数 MB → 数百万文字）を防ぐ
4. **縮退の明示** — 未接続 MCP の項目は「未実施」と明記
5. **二層応答** — 機械可読サマリ + 人間向けレポート

## 同梱 Skill の同期について

`skills/` 配下は [pdf-trust-skill](https://github.com/shuji-bonji/pdf-trust-skill) /
[pdf-publish-skill](https://github.com/shuji-bonji/pdf-publish-skill) からの**コピー同梱**。
原本は各リポジトリであり、更新時は以下で再同期する:

```bash
cp -R ../pdf-trust-skill/skills/pdf-trust skills/
cp -R ../pdf-publish-skill/skills/pdf-publish skills/
```

## 既知の未検証事項（実運用後に補正）

- [ ] `tools` のワイルドカード（`mcp__pdf-reader__*` 等）が、プラグイン同梱 mcpServers の
      実際のツール名プレフィックスと一致するか（環境によっては
      `mcp__plugin_..._pdf-reader__*` 形式になる可能性）
- [x] plugin.json の `env` での `${VAR}` 展開は**機能しない**（2026-07-26 に別プラグインの
      実挙動で確認: `"${PDF_SPEC_DIR}"` がリテラルのまま渡り REGISTRY_ERROR）。
      → env ブロックを外し、シェル環境（launchd / .zshenv）の継承に任せる方式に変更済み
- [ ] 委譲の発火精度（description の文言調整）
- [ ] model 指定（sonnet で十分か。仕様相談の比重が高ければ上位系へ）
- [ ] 複数 PDF 並列監査時の挙動

## ライセンス

MIT
