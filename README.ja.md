# pdf-specialist-plugin

[English](./README.md)

PDF 専門家サブエージェント **pdf-specialist** の統合プラグイン。
1 プラグインで「エージェント定義 + PDF Family 4 MCP サーバ + pdf-trust / pdf-publish Skill」が揃う（MCP・Skill とも依存宣言で自動導入）。

> 設計書: PDF 専門家エージェント設計書 パターン1（Claude サブエージェント構成）の実装。
> **現状 v0.4.0 も未実運用**。実行利用の結果で残る詳細（model・発火条件）を補正する。

## 構成

```
pdf-specialist-plugin/
├── .claude-plugin/plugin.json   # マニフェスト + dependencies
└── agents/pdf-specialist.md     # サブエージェント定義（委譲トリガー + 絶対規則）
```

**同梱物は無い。** MCP 4 サーバも Skill 2 つも `dependencies` で宣言し、同じ marketplace から
自動で入る（下記「すべて依存で入る」）。

## インストール

```
/plugin marketplace add shuji-bonji/claude-plugins
/plugin install pdf-specialist
```

依存 6 件（`pdf-trust` / `pdf-publish` + MCP プラグイン 4 つ）は自動で入り、install 出力の末尾に
列挙される。**Claude Code v2.1.110 以上**が必要（enable / disable が依存へ伝播するのは v2.1.143 以上）。

### 必要な環境設定

| 変数 | 対象 | 必須 | 内容 |
|---|---|---|---|
| `PDF_SPEC_DIR` | pdf-spec | pdf-spec を使うなら必須 | 仕様 PDF コーパスのディレクトリ。未設定だと pdf-spec は起動失敗するが、他 3 サーバと Skill は動作する |
| `PDF_VERIFY_VERAPDF` | pdf-verify | 任意 | veraPDF 実行パス。無ければ PATH 探索 → 内蔵ルールに縮退 |
| `PDF_VERIFY_TRUST_ANCHORS` | pdf-verify | 任意 | 信頼アンカー証明書ディレクトリ。無いと verdict は use_with_caution 止まり |
| `PDF_WRITER_FONT` | pdf-writer | 日本語出力に実質必須 | 単一フェイス .ttf/.otf（Noto Sans JP static 版推奨） |

環境変数はシェル環境（launchd / .zshenv 等）に設定する。サーバは MCP プラグイン 4 つが提供し、起動時にその環境を継承する。

### バージョン要件

- pdf-verify-mcp **v0.7.0+**（evaluate_policy。pdf-trust の 4 値判定に必須）／
  **v0.10.0+ 推奨**（`verify_integrity` のリビジョン間オブジェクト単位差分。
  「署名後に何が変わったか」をオブジェクト単位で言えるようになる。4 値判定自体は不変）
- pdf-reader-mcp **v0.10.0+** 推奨（`locate_objects` = オブジェクト番号 → ページ + 矩形。
  上の差分を位置に落とすのに要る）／**v0.11.0+** で逆方向も揃う
  （`extract_structured_text` の `include_bbox` = **構造要素** → 矩形。
  「この段落に注釈」も人間が座標を指定せずに済む）
- pdf-writer-mcp **v0.15.0+** 推奨（preserveSignatures / tag_form_fields）。
  **v0.16.0+** で PDF 2.0 出力と PDF/A-4 / PDF/A-4f の器付け（`ensure_pdfa` の `flavour`）が入った

各 MCP プラグインが `@latest` で接続するため通常は満たされる。`dependencies` にバージョン範囲は固定していない（下記）。

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

## すべて依存で入る

```json
"dependencies": [
  "pdf-trust", "pdf-publish",
  "pdf-reader-mcp", "pdf-spec-mcp", "pdf-verify-mcp", "pdf-writer-mcp"
]
```

このプラグインが持つのはエージェント定義だけ。2 段階でそうなった。**どちらも原因は同じ**。

**v0.3.0 — Skill。** v0.2.0 までは `skills/` への**コピー同梱**で、[pdf-trust-skill](https://github.com/shuji-bonji/pdf-trust-skill) /
[pdf-publish-skill](https://github.com/shuji-bonji/pdf-publish-skill) から手で `cp -R` していた。
コピーは古くなる — 実際になった。install 済みプラグインは**自分のディレクトリの外を参照できない**
（symlink も submodule も install 後のキャッシュに乗らない）ので、選択肢は「同梱」か「依存」の
二択であり、ズレようがないのは後者。

**v0.4.0 — MCP サーバ。** マニフェストに 4 サーバを直書きしていたため、単体の MCP プラグインと
**同じコマンドだと判定されて skip** されていた（`/plugin` の Errors タブに出る）。さらに悪いことに、
MCP のツール名は `mcp__plugin_<プラグイン名>_<サーバ名>__<ツール名>` なので、
**プレフィックスが「重複判定でどちらが勝ったか」= 利用者が他に何を入れているかで変わる**。
静的な `tools` allowlist で両方に正しく書くことは不可能。依存宣言にすれば名前が一意に決まる。

これは見た目より重い問題だった: v0.3.0 までエージェントの allowlist は `mcp__pdf-reader__*` で、
公式の命名規則ではこれは**何にもマッチしない**。つまり **エージェントは PDF ツールを 1 つも
持っていなかった**。v0.4.0 で `agents/pdf-specialist.md` に完全なプレフィックスを書いた。

6 つとも**このプラグインと同じ marketplace 内**で解決されるため、
`allowCrossMarketplaceDependenciesOn` の登録は不要。バージョン範囲は固定していない
（各依存は marketplace エントリが提供する版に追随する）。固定するには marketplace 側に
`pdf-trust--v{version}` 形式の git タグが要り、これは現行のリポジトリ単位のリリースタグとは別の規約になる。

## 既知の未検証事項（実運用後に補正）

- [x] **依存解決は実測で動作**（2026-07-27・Claude Code v2.1.212）。クリーン状態からの
      `claude plugin install pdf-specialist@shuji-bonji` が `(+ 2 dependencies: pdf-publish, pdf-trust)`
      を報告／uninstall では pdf-trust が「自動導入された依存」として `claude plugin prune` の
      対象に挙がる（＝手動導入と区別されている）／pdf-specialist が有効な状態で pdf-trust を
      disable しようとすると拒否され、正しい順序のコマンドが提示される。
      **未検証なのは委譲の発火と日常運用の方**
- [x] **`tools` のワイルドカードは一致していなかった**（2026-07-27 発見）。公式の命名規則は
      `mcp__plugin_<プラグイン名>_<サーバ名>__<ツール名>` なので `mcp__pdf-reader__*` は
      何にもマッチせず、allowlist の結果 **エージェントは Read / Glob / Skill しか持っていなかった**。
      v0.4.0 で完全なプレフィックスを明記し、MCP を依存へ移して名前が環境に依存しないようにした
- [x] plugin.json の `env` での `${VAR}` 展開は**機能しない**（2026-07-26 に別プラグインの
      実挙動で確認: `"${PDF_SPEC_DIR}"` がリテラルのまま渡り REGISTRY_ERROR）。
      → env ブロックを外し、シェル環境（launchd / .zshenv）の継承に任せる方式に変更済み
- [ ] 委譲の発火精度（description の文言調整）
- [ ] model 指定（sonnet で十分か。仕様相談の比重が高ければ上位系へ）
- [ ] 複数 PDF 並列監査時の挙動

## ライセンス

MIT
