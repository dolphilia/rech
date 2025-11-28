# HTML/JS

---

## 0. 前提・全体像

* WebView は `file://...` で `index.html` または `pages/*.html` を開く。

* help.pkg の中身は（前回の案）：

  ```text
  /meta.json
  /topics.json
  /search_index.json
  /index.html
  /pages/*.html
  /assets/css/*.css
  /assets/js/main.js
  ```

* 検索に必要なのは：

  * `search_index.json`：検索用インデックス
  * `topics.json`：検索結果から topic_id を URL に変換するため（クライアント側でも利用）

Rust 側は「最初にどのページを開くか」まで担当し、
それ以降のページ遷移・検索は **HTML/JS 側で完結**させるイメージです。

---

## 1. HTML のざっくり構造

### 1-1. 共通レイアウト

`index.html` / `pages/*.html` のどちらから開かれても同じレイアウトになるよう、
共通のヘッダ＆検索 UI を用意します。

```html
<body>
  <header class="hv-header">
    <input id="hv-search-input" type="search" placeholder="ヘルプを検索…" />
    <div id="hv-search-results" class="hv-search-results"></div>
  </header>

  <main class="hv-main">
    <!-- （ここに各ページ固有の本文） -->
    <article class="hv-content">
      <!-- Markdown → HTML の本体 -->
    </article>
  </main>

  <script src="../assets/js/main.js"></script> <!-- pages/ 下の場合 -->
</body>
```

※ `index.html` では `<script src="assets/js/main.js">` のように、相対パスだけ変える想定。
（この辺はビルド時に自動で差し替える、などでもOK）

### 1-2. 検索結果表示エリア

`#hv-search-results` に検索結果のリストを描画します：

* 最大 N 件（例えば 20 件）
* 各行は：タイトル＋サマリ＋（オプションで）パス情報
* クリックで該当 topic に遷移

---

## 2. `search_index.json` の仕様（HTML/JS 視点）

再掲＋少し JS 寄りに整理します。

```jsonc
{
  "entries": [
    {
      "topic_id": "getting-started",
      "title": "はじめに",
      "summary": "MyApp の概要と基本的な使い方について説明します。",
      "keywords": ["はじめに", "概要", "イントロ"],
      "content": "MyApp の概要と基本的な使い方について…"
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

* `topic_id`：

  * 結果クリック時に `topics.json` から URL を求めるためのキー。
* `title`：

  * 検索結果リストのタイトル表示に使用。
* `summary`：

  * リストのサブテキストとして表示。
* `keywords`：

  * 簡易検索時の重みづけ用（あれば）。
* `content`：

  * 簡易実装では `includes` / 正規表現検索に使う全文。長すぎる場合はビルド時にトリミング。

初期バージョンでは **「単純な部分一致検索」** を想定しておき、
将来的に Lunr/FlexSearch 等を導入する余地を残します。

---

## 3. `topics.json` のクライアント側利用

`topics.json`（Rust 側と共通仕様）：

```jsonc
{
  "topics": [
    {
      "id": "getting-started",
      "title": "はじめに",
      "page": "getting-started",
      "anchor": null,
      "tags": []
    },
    {
      "id": "install.windows",
      "title": "インストール（Windows）",
      "page": "install",
      "anchor": "windows",
      "tags": ["Windows"]
    }
  ]
}
```

HTML/JS側ではこれを使って：

* `topic_id` → `pages/<page>.html#<anchor>` という **相対URL** を組み立てる。
* 検索結果クリック時に `navigateToTopic(topic_id)` を呼び出し、
  `window.location.href = 相対URL` で遷移。

検索以外でも、左ペインの TOC（将来）などはこの `topics.json` を共通で使えます。

---

## 4. JavaScript の役割分担

1ファイル `assets/js/main.js` の中を大きく三つに分けて考えます：

1. **初期化**：DOM準備・イベントハンドラ登録
2. **検索インデックス管理**：`search_index.json` のロード＆メモリ保持
3. **トピックナビゲーション**：`topic_id` → URL → `location.href` の処理

### 4-1. 初期化フェーズ

* DOMContentLoaded 時に以下を行う：

  1. `#hv-search-input` があれば `input` イベントのハンドラを登録

     * ここで「入力が一定時間止まったら検索」を行うための **簡易デバウンス** を実装
  2. 検索の初回実行時に `search_index.json` をロード（Lazy Load）

### 4-2. search_index.json のロード

`file://` でも `fetch()` は使える前提ですが、
相対パスを決めるために **「ページから見た search_index.json の位置」**をルール化しておきます。

* ルール案：

  * `index.html` から見たパス：`./search_index.json`
  * `pages/*.html` から見たパス：`../search_index.json`

これを JS 側で判定する簡易仕様：

* `window.location.pathname` を見て、

  * `/pages/` を含んでいれば `../search_index.json`
  * そうでなければ `./search_index.json`

※ ビルドパイプラインで `<script>` 内に埋め込む形でも良いですが、仕様としては上記のような判定ルールを決めておけば十分です。

### 4-3. 検索ロジック（簡易版仕様）

* 検索文字列 `q` が空のとき：

  * 検索結果エリアを空にする or 非表示
* 空でないとき：

  1. `entries` を線形走査
  2. 大文字小文字を無視するなら両方小文字化
  3. 判定に使うフィールド：

     * `title`
     * `keywords`（配列）
     * `summary`
     * `content`（必要なら）
  4. `includes(q)` で一致した entry を集める
  5. スコアリング（簡易）：

     * タイトルに含まれる場合：+3
     * キーワードに含まれる場合：+2
     * サマリ：+1
     * 本文：+0.5
  6. スコア順に sort して上位 N 件を表示

→ 仕様としては「**単純一致 ＋ 簡易スコアリング**」まで決めておき、
将来インデックス形式を変えたい時には search_index.json の形式ごと差し替える想定です。

### 4-4. 検索結果の表示仕様

* `#hv-search-results` 内に `<ul>` / `<li>` で描画：

  ```html
  <ul>
    <li data-topic-id="install.windows">
      <div class="hv-result-title">インストール（Windows）</div>
      <div class="hv-result-summary">Windows へのインストール手順。</div>
    </li>
    ...
  </ul>
  ```

* `li` クリック時：

  * `data-topic-id` を読み取り
  * `navigateToTopic(topicId)` を呼んでページ遷移
  * 遷移後、検索結果エリアは自動的に消える（仕様としては「遷移前に消す」でOK）

---

## 5. `navigateToTopic(topicId)` の仕様

### 5-1. topics.json のロード

* ページ初期化時または初回 `navigateToTopic` 呼び出し時に `topics.json` をロード

* こちらも `search_index.json` と同じくパスを決める：

  * `index.html` → `./topics.json`
  * `pages/*.html` → `../topics.json`

* ロード後、`topicMap: { [id: string]: { page: string, anchor?: string } }` をメモリに保持

### 5-2. ルーティング仕様

`topicId` を受け取ったら：

1. `topicMap[topicId]` を探す。なければ何もしない（将来はエラー表示）。

2. let `page = entry.page`, `anchor = entry.anchor`

3. URL を組み立て：

   * 基本形：`url = "pages/" + page + ".html"`
   * `anchor` があれば `url += "#" + anchor`

4. 現在のパス位置から見て相対 URL にするルール：

   * もし今が `index.html` の場合：そのまま `url` を使う
   * もし今が `pages/*.html` の場合：`url` の前に `"../"` を付ける

5. `window.location.href = url` で遷移

※ ここを SPA 的に `pushState` で差し替える案もありますが、
まずは「普通のページ遷移」で十分だと思います（軽くてシンプル）。

---

## 6. エラー／フェイルセーフ仕様

* `search_index.json` / `topics.json` のロードに失敗した場合：

  * コンソールにエラーを出す。
  * 検索 UI は「無効化」する（入力できても結果が出ない or エラーメッセージ表示）。
* `topicId` が見つからない場合：

  * 何も起きない or 簡易なアラートを出す（初期版では何もしなくてもよい）。

---

## 7. ロードタイミング・パフォーマンス方針

* `search_index.json` と `topics.json` は **初回利用時に Lazy Load**：

  * ページ表示時にいきなり全部ロードしない（ヘルプ閲覧だけなら不要なため）
* 1ページ内ではキャッシュ：

  * ページ遷移ごとに再ロードされるが、初期版は割り切り
  * 将来 SPA 化や Service Worker を導入する余地は残しておく

---

## まとめ

HTML/JS 側の簡易仕様としては：

* ヘッダ部分に検索入力＋結果エリアを共通実装
* `search_index.json` をクライアント側 JS でロードし、
  単純な部分一致＋簡易スコアリングで検索
* `topics.json` を使って `topic_id → pages/<page>.html#<anchor>` に変換し、
  `window.location.href` で遷移
* どちらの JSON も「相対パスのルール」をあらかじめ決めておく

という形になります。