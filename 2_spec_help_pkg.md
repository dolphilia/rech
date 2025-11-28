# help.pkg の仕様

## 0. 全体ディレクトリ構造（案）

まず ZIP（help.pkg）の中身を、こんな感じの構成にします：

```text
help.pkg (ZIP)

  /meta.json            ... メタデータ（バージョン・言語・ビルド情報など）
  /toc.json             ... 目次ツリー（論理構造）
  /topics.json          ... topic ID → ページ/アンカー対応表
  /search_index.json    ... 検索インデックス（軽量なJSON）
  /index.html           ... ヘルプトップページ
  /pages/
      <page-id>.html    ... 各ページ（chapter, sectionなど）
  /assets/
      css/*
      js/*
      img/*
```

**ポイント**

* `topics.json` を設けて、「アプリ側から指定される topic ID」と「実際の HTML ファイル＋アンカー」を分離します。
* `toc.json` はあくまで「表示用のツリー構造」。`topics.json` を参照して URL に落とします。
* `search_index.json` は JS 側が読む前提（Lunr/FlexSearch 等）。

---

## 1. `meta.json` スキーマ

ヘルプ全体のメタ情報を持つファイルです。
最低限＋拡張余地を含めてこんな感じ：

```jsonc
{
  "format_version": 1,
  "package_id": "com.example.myapp.help",
  "title": "MyApp ヘルプ",
  "description": "MyApp のオフラインヘルプ",
  "language": "ja-JP",
  "app_version": "1.2.3",
  "help_version": "2025.01",
  "build_timestamp": "2025-11-28T12:34:56Z",

  "default_page": "index",             // topics.json の topic_id または page_id
  "default_topic": "getting-started",  // 任意（あればこちら優先）

  "generator": {
    "name": "myapp-doc-builder",
    "version": "0.3.0"
  },

  "options": {
    "search_enabled": true,
    "toc_enabled": true,
    "dark_mode_supported": true
  }
}
```

### 意図

* `format_version`：help.pkg の仕様バージョン（Breaking 変更の判定用）
* `app_version`：ヘルプの対象アプリバージョン
* `help_version`：ヘルプ側のバージョン管理（文字列でOK）
* `language`：`ja-JP`, `en-US` など。将来多言語パッケージを用意する基準に。

---

## 2. `topics.json` スキーマ

**アプリ側が「topic ID」でヘルプを指定**できるようにするマッピング表です。

### コンセプト

* topic ID は 「ドメイン名っぽい階層表現」を推奨：

  * `getting-started`
  * `install.windows`
  * `editor.shortcuts`
  * `settings.display.theme`
* 1 topic = 1 HTML ページの任意のアンカーに対応

  * 例：`pages/getting-started.html#installation-windows`

### スキーマ例

```jsonc
{
  "topics": [
    {
      "id": "getting-started",
      "title": "はじめに",
      "page": "getting-started",        // pages/getting-started.html
      "anchor": null,                   // null または "" ならページ先頭
      "tags": ["intro", "overview"]     // 検索補助用の軽いタグ
    },
    {
      "id": "install.windows",
      "title": "インストール（Windows）",
      "page": "install",
      "anchor": "windows",
      "tags": ["install", "windows"]
    },
    {
      "id": "editor.shortcuts",
      "title": "エディタショートカット",
      "page": "editor",
      "anchor": "shortcuts",
      "tags": ["shortcuts", "keymap", "editor"]
    }
  ]
}
```

### ビューア側の利用方法

1. `--topic editor.shortcuts` で起動されたら `topics.json` を読む
2. `id == "editor.shortcuts"` を探す
3. URL を `pages/<page>.html#<anchor>` 形式で組み立てる

   * anchor が null/空なら `pages/editor.html`
   * anchor が `"shortcuts"` なら `pages/editor.html#shortcuts`
4. WebView に `file:///.../pages/editor.html#shortcuts` をロード

### 注意事項

* `id` は **pkg 内で一意**であること。
* `page` は拡張子なしの ID とし、ファイル名は `page + ".html"` に統一。
* これにより、ビルドスクリプトとビューア両方が単純になる。

---

## 3. `toc.json` スキーマ（目次ツリー）

**表示用のツリー構造**。topic ID を参照して論理構造を表現します。

### スキーマ例

```jsonc
{
  "root_items": [
    {
      "title": "はじめに",
      "topic_id": "getting-started",
      "children": [
        {
          "title": "概要",
          "topic_id": "getting-started",
          "children": []
        },
        {
          "title": "インストール",
          "topic_id": null,
          "children": [
            {
              "title": "Windows",
              "topic_id": "install.windows",
              "children": []
            },
            {
              "title": "macOS",
              "topic_id": "install.macos",
              "children": []
            }
          ]
        }
      ]
    },
    {
      "title": "エディタ",
      "topic_id": null,
      "children": [
        {
          "title": "ショートカット",
          "topic_id": "editor.shortcuts",
          "children": []
        }
      ]
    }
  ]
}
```

### ポイント

* トピックを持たない「見出しノード」（topic_id = null）もOK。
* 1 ノードに `title` と `topic_id` と `children` があれば十分。
* ビューア側はこのツリーを左ペインに表示し、クリック時に `topic_id` を解決して WebViewで遷移。

---

## 4. `search_index.json` スキーマ（軽量版）

ここは戦略次第ですが、**まずは単純な構造**から始めるのをおすすめします。

### パターンA：JSライブラリ（Lunr 等）用のシリアライズ形式

→ ライブラリに依存するので、仕様確定には少し重い。

### パターンB：素朴な自前検索用のシンプルな構造（初期案）

```jsonc
{
  "entries": [
    {
      "topic_id": "getting-started",
      "title": "はじめに",
      "summary": "MyApp の概要と基本的な使い方について説明します。",
      "keywords": ["はじめに", "概要", "intro"],
      "content": "MyApp の概要と基本的な使い方について…（適度にトリミング）"
    },
    {
      "topic_id": "install.windows",
      "title": "インストール（Windows）",
      "summary": "Windows へのインストール手順。",
      "keywords": ["インストール", "Windows", "セットアップ"],
      "content": "Windows へのインストール手順として…"
    }
  ]
}
```

### 利用方針

* **ステップ1（最初の実装）**

  * JSで `entries` を読み込み、全文を単純に `includes` / 正規表現で検索
  * まずは件数も少ない前提で、実装をシンプルにする

* **ステップ2（必要になれば）**

  * 検索インデックスを事前構築（N-gram / Tokenize）
  * Lunr.js や FlexSearch のインデックス構造で保存する

ビューア仕様書では、まず「検索API」として

* `topic_id`
* `title`
* `summary`
* `keywords`
* `content`（オプション）

あたりを持つ構造にしておけば、裏側のアルゴリズムは差し替え可能です。

---

## 5. HTML 側の構成と制約

### 基本ルール

* `index.html`：トップページ
* `pages/<page-id>.html`：各論理ページ
* それぞれ `id` 属性でアンカーを打つ：

```html
<h2 id="shortcuts">ショートカット一覧</h2>
<p>…</p>
```

* `topics.json` の `page` + `anchor` は **この`id`を参照**する。

### 推奨する書き方（生成を想定）

* コンテンツは Markdown から生成（mdBook / Sphinx など）
* 生成時に「見出しID生成ルール」を固定（例：`slugify(title)`）

---

## 6. 多言語・バージョン展開の設計余地

今後、多言語化やバージョン違いの help.pkg を切り替えたい場合に備えて、`meta.json` に最低限これを持ちます：

```jsonc
{
  "language": "ja-JP",
  "app_version": "1.2.3",
  "help_version": "2025.01",
  "aliases": ["ja", "ja_JP"]
}
```

ビューアアプリの側では：

* アプリ本体から「想定言語」「アプリバージョン」を教えてもらう
* 適切な help.pkg を選ぶ（複数設置されていた場合）

…という拡張が将来可能です。
（この辺りは help.pkg の外の仕様ですが、中側の `meta.json` は今の段階で準備しておくと楽です）

---

## 7. 例：最小構成 help.pkg の中身イメージ

「最初のプロトタイプ用に、本当に最小限だけに削った例」です：

```text
help.pkg
  /meta.json
  /topics.json
  /index.html
  /pages/getting-started.html
  /pages/install.html
  /assets/css/style.css
  /assets/js/main.js
```

`meta.json`

```json
{
  "format_version": 1,
  "package_id": "example.myapp.help",
  "title": "MyApp ヘルプ",
  "language": "ja-JP",
  "app_version": "0.1.0",
  "help_version": "2025.01",
  "build_timestamp": "2025-11-28T12:00:00Z",
  "default_page": "getting-started",
  "generator": {
    "name": "handcrafted",
    "version": "0.0.1"
  },
  "options": {
    "search_enabled": false,
    "toc_enabled": false,
    "dark_mode_supported": false
  }
}
```

`topics.json`

```json
{
  "topics": [
    {
      "id": "getting-started",
      "title": "はじめに",
      "page": "getting-started",
      "anchor": null,
      "tags": []
    }
  ]
}
```

これだけあれば、

* `help-viewer --topic getting-started`
* → topics.json を見て `pages/getting-started.html` を開く

という最低限の動作が実現できます。
