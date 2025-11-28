# 📄 **アプリ仕様書（ドラフト）**

## *Help Viewer — Rust + wry を用いた軽量クロスプラットフォームヘルプアプリ*

---

## 1. 目的

本アプリは、`help.pkg`（ZIP 形式）に格納されたヘルプドキュメントを
**Windows / macOS / Linux** 上で軽量に閲覧することを目的とする。

特長：

| 目標          | 内容                                 |
| ----------- | ---------------------------------- |
| 軽快な動作       | Chromiumを同梱せずOSネイティブWebViewを使用     |
| クロスプラットフォーム | Rust + wry (WebView wrapper) で統一対応 |
| オフライン       | 全データは help.pkg 内に収まり、外部接続不要        |
| 単一ツール配布     | 可能なら単一バイナリ化（依存最小）                  |
| CHM代替の現代的方式 | HTMLベースのヘルプをアプリ内ビューアとして提供          |

---

## 2. 仕様概要

### ▷ 主構成

```
+---------------------------------------------+
| Help Viewer (Rust + wry)                    |
|                                             |
|  Startup: read help.pkg → extract → serve   |
|  Display: OS-native WebView                 |
+---------------------------------------------+

help.pkg = ZIP
    /index.html
    /pages/*.html
    /search_index.json
    /assets/*
```

---

## 3. help.pkg 仕様（初期案）

| パス                   | 内容                           |
| -------------------- | ---------------------------- |
| `/index.html`        | ヘルプトップページ                    |
| `/pages/*.html`      | 文書本体（章ごとなど）                  |
| `/search_index.json` | ローカル検索用インデックス（Lunr.js/自前構成可） |
| `/assets/*`          | CSS, JS, 画像, アイコン等           |

拡張余地：

* `/toc.json`: 目次構造を独自定義し、ビューア側から読み出せるようにする
* `/meta.json`: バージョン情報／生成日時／検索設定／表示テーマなど

---

## 4. アプリ基本機能

| 機能            | 内容                                |
| ------------- | --------------------------------- |
| help.pkg 読み込み | ZIP読み込み→一時展開                      |
| HTMLレンダリング    | wry 経由で OS WebView を利用            |
| ページ間リンク       | HTML anchor (`#/topic=xxx`) をサポート |
| 検索機能          | search_index.json をJS側で検索         |
| コンテキストジャンプ    | `--topic xxx` により該当ページへ直接遷移       |
| オフライン動作       | ネット接続不要で閲覧可                       |

---

## 5. 起動仕様・コマンドライン

```
help-viewer [options]

Options:
  --pkg <path>        読み込む help.pkg の指定（省略時は同ディレクトリ検索）
  --topic <id>        指定トピックを開く（例: editor.shortcuts）
  --extract-only      展開のみ実行しWebView生成しない（デバッグ用）
```

実行例：

```
help-viewer --pkg ./docs/help.pkg
help-viewer --pkg /opt/app/help.pkg --topic install.windows
```

---

## 6. 画面仕様（GUIスケッチ）

```
+--------------------------------------------------------+
|  [🔍 Search box]                                       |
|--------------------------------------------------------|
|  TOC (optional)            |   WebView (HTML表示)      |
|  - Getting Started         |   pages/.. / index.html   |
|  - Installation            |                          |
|  - Shortcuts               |                          |
+--------------------------------------------------------+
|  Status bar (ページURL/検索ヒット件数等)               |
+--------------------------------------------------------+
```

TOC（目次）は初期版では省略可能。
将来 `/toc.json` 追加時に UIとして拡張。

---

## 7. 使用技術

| 階層       | 採用技術                            |
| -------- | ------------------------------- |
| 言語       | **Rust**                        |
| UI/ウィンドウ | tao                             |
| WebView  | **wry**（OS依存のWebViewを利用）        |
| ZIP処理    | `zip` / `flate2` / `zip-rs`     |
| HTML側検索  | Lunr.js / FlexSearch / 自前JSON走査 |

OS WebView 使用対象：

| OS      | WebViewエンジン        |
| ------- | ------------------ |
| Windows | WebView2           |
| macOS   | WKWebView          |
| Linux   | WebKitGTK（導入前提要確認） |

---

## 8. 非機能要件

| 項目   | 目標                      |
| ---- | ----------------------- |
| 起動時間 | 1秒以内（展開済みキャッシュ前提）       |
| メモリ  | 100MB以下を目標（WebView含む）   |
| 容量   | バイナリ＋pkg含め **20MB以下推奨** |
| 移植性  | 3OS全対応、CIビルド可能          |

---

## 9. 今後の拡張案

| 拡張                 | 概要           |
| ------------------ | ------------ |
| DarkMode対応         | CSSテーマ切替     |
| Favourite/Bookmark | ページのスター登録    |
| 履歴機能               | 戻る/進むの実装     |
| help.pkg暗号化        | 商用アプリ向け保護    |
| コンテキストAPI          | 本体アプリとのIPC連携 |
