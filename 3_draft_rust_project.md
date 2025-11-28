# Rust プロジェクト構成

## 1. プロジェクト構成（ディレクトリ・モジュール）

シンプルな単一バイナリ crate を想定します。

```text
help-viewer/
  Cargo.toml
  src/
    main.rs
    cli.rs           // コマンドライン引数解析
    config.rs        // 設定・パスなど
    pkg/
      mod.rs         // help.pkg ハンドリング
      meta.rs        // meta.json の構造体
      topics.rs      // topics.json の構造体 & 解決ロジック
      toc.rs         // toc.json（将来拡張）
    extract.rs       // ZIP 展開ロジック
    app/
      mod.rs         // アプリ全体の起動フロー（状態管理）
      webview.rs     // wry + tao によるウィンドウ／WebView 管理
      router.rs      // topic_id → URL 生成
      state.rs       // 実行時状態（現在の pkg パス、展開先など）
```

### 各モジュールの役割（ざっくり）

* `main.rs`

  * エントリポイント。`cli` で引数解析 → `app::run()` に制御移譲
* `cli.rs`

  * `--pkg` / `--topic` / `--extract-only` などを解析
* `config.rs`

  * デフォルトの help.pkg パス、キャッシュディレクトリ決定ロジック
* `pkg::*`

  * help.pkg 内の JSON（`meta.json` / `topics.json` …）の読み込みとパース
* `extract.rs`

  * ZIP 展開処理。展開先ディレクトリの管理（キャッシュ再利用など）
* `app::*`

  * アプリ全体の制御・状態
  * WebView の生成・URLナビゲーション
  * topic_id からページURLへのルーティング

---

## 2. 使用クレート（想定）

仕様段階での候補（実際には変えるかもしれませんが、イメージとして）：

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }     # cli.rs用
serde = { version = "1", features = ["derive"] }    # JSON構造体
serde_json = "1"
zip = "0.6"                                        # ZIP展開
anyhow = "1"                                       # エラーまとめ
thiserror = "1"                                    # エラー定義（任意）

tao = "0.27"                                       # ウィンドウ・イベントループ
wry = "0.43"                                       # WebView

# ファイル/パス関連
dirs = "5"                                         # OSごとのパス決定 (cache dir 等)
```

※ バージョンはイメージ。実際には最新安定版を指定する前提。

---

## 3. 起動〜表示までのフロー（高レベル）

### シーケンス（ざっくり）

1. `main.rs`

   * `cli::parse_args()` で引数解析
   * `config::resolve_pkg_path()` で help.pkg のパスを決定
   * `app::run(AppParams { pkg_path, topic })` を呼び出し
2. `app::run`

   * `extract::prepare_pkg(&pkg_path)` で展開ディレクトリを取得
   * `pkg::load_meta()`, `pkg::load_topics()` を呼ぶ
   * `app::state::AppState` を初期化
   * `app::webview::run_event_loop(state)` を呼ぶ
3. `webview::run_event_loop`

   * `tao::EventLoop` と `wry::WebView` を構築
   * 初期表示URLを `router::resolve_initial_url(&state, topic)` で決定
   * WebView に `file:///.../index.html` または `pages/...#anchor` をロード
   * イベントループ開始

---

## 4. 各モジュール詳細仕様

### 4.1 cli.rs

```rust
/// コマンドラインオプション
pub struct CliArgs {
    pub pkg_path: Option<PathBuf>,
    pub topic: Option<String>,
    pub extract_only: bool,
}

pub fn parse_args() -> CliArgs;
```

* `pkg_path`: `--pkg` 明示指定がない場合は `None`
* `topic`: `--topic` が指定された場合に `Some("editor.shortcuts".into())`
* `extract_only`: true の場合、`app::run` ではなく `extract::prepare_pkg` までで終了するデバッグ用

---

### 4.2 config.rs

```rust
pub struct ResolvedConfig {
    pub pkg_path: PathBuf,
}

pub fn resolve_pkg_path(cli: &CliArgs) -> anyhow::Result<ResolvedConfig>;
```

ロジック例：

1. `cli.pkg_path` があればそれを使う
2. なければ

   * カレントディレクトリの `./help.pkg`
   * 実行ファイルの隣の `help.pkg`
   * OS固有の `resources/help.pkg` 等
     を順に探索し、最初に見つかったものを採用

---

### 4.3 extract.rs（help.pkg 展開）

```rust
/// 展開先に関する情報
pub struct ExtractedPkg {
    pub root_dir: PathBuf,  // 展開されたルートディレクトリ
}

pub fn prepare_pkg(pkg_path: &Path) -> anyhow::Result<ExtractedPkg>;
```

仕様（簡易版）：

* キャッシュ機構は最初は無しでも良い（毎回 tmp に展開）
* 将来的には：

  * `~/.cache/help-viewer/<hash(pkg_path + help_version)>/` に展開
  * 既存ディレクトリがあれば再展開を省略

展開後、少なくとも以下のファイルが存在することを確認：

* `<root>/meta.json`
* `<root>/topics.json`
* `<root>/index.html` または `<root>/pages/*.html`

条件を満たさなければエラーを返す。

---

### 4.4 pkg::meta.rs / topics.rs

#### meta.rs

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct Meta {
    pub format_version: u32,
    pub package_id: String,
    pub title: String,
    pub description: Option<String>,
    pub language: String,
    pub app_version: Option<String>,
    pub help_version: Option<String>,
    pub build_timestamp: Option<String>,
    pub default_page: Option<String>,
    pub default_topic: Option<String>,
    pub options: Option<MetaOptions>,
}

#[derive(Debug, Deserialize)]
pub struct MetaOptions {
    pub search_enabled: Option<bool>,
    pub toc_enabled: Option<bool>,
    pub dark_mode_supported: Option<bool>,
}

pub fn load_meta(root: &Path) -> anyhow::Result<Meta>;
```

#### topics.rs

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct TopicEntry {
    pub id: String,
    pub title: String,
    pub page: String,
    pub anchor: Option<String>,
    pub tags: Option<Vec<String>>,
}

#[derive(Debug, Deserialize)]
pub struct Topics {
    pub topics: Vec<TopicEntry>,
}

pub fn load_topics(root: &Path) -> anyhow::Result<Topics>;

impl Topics {
    pub fn find(&self, id: &str) -> Option<&TopicEntry> {
        self.topics.iter().find(|t| t.id == id)
    }
}
```

---

### 4.5 app::state.rs

```rust
pub struct AppState {
    pub extracted: ExtractedPkg,
    pub meta: Meta,
    pub topics: Topics,

    pub initial_topic: Option<String>, // cli から渡されたもの
}

impl AppState {
    pub fn root_dir(&self) -> &Path {
        &self.extracted.root_dir
    }
}
```

---

### 4.6 app::router.rs（topic → URL）

```rust
use crate::pkg::topics::Topics;
use crate::pkg::meta::Meta;

pub fn resolve_initial_url(
    root_dir: &Path,
    topics: &Topics,
    meta: &Meta,
    cli_topic: &Option<String>,
) -> anyhow::Result<Url>;
```

ロジック：

1. `cli_topic` が `Some(id)` なら `topics.find(id)` を試す

   * 見つかったら `file:///root_dir/pages/{page}.html[#anchor]`
2. 見つからない / `None` の場合：

   * `meta.default_topic` があれば同様に解決
   * なければ `meta.default_page` を使って `pages/{page}.html`
   * それもなければ `root_dir/index.html` を開く

`Url` は `wry` に渡せる `http` or `file` URL。
ここでは `file://` を使う想定。

---

### 4.7 app::webview.rs（wry + tao）

仕様レベルでのイメージ：

```rust
pub fn run_event_loop(state: AppState) -> anyhow::Result<()> {
    // 1. EventLoop の生成
    let event_loop = EventLoop::new();

    // 2. WindowBuilder の設定（タイトル = meta.title 等）
    let window_builder = WindowBuilder::new()
        .with_title(state.meta.title.clone());

    // 3. WebViewBuilder の設定
    let initial_url = router::resolve_initial_url(...)?;

    let webview = WebViewBuilder::new(window_builder)?
        .with_url(initial_url.as_str())?
        // 必要なら JS との IPC 設定などもここで
        .build(&event_loop)?;

    // 4. イベントループ起動
    event_loop.run(move |event, _, control_flow| {
        // 閉じるボタンで ControlFlow::Exit 等
    });
}
```

詳細は実装段階で詰めるとして、仕様としては：

* Window タイトルは基本 `meta.title` を使用
* 画面初期サイズは 900x600 などのデフォルト値（将来設定保存可）
* 閉じるとプロセス終了
* 戻る/進むなどは初期版では未実装でもよい（キーボードショートカットでそのうち）

---

## 5. ファイル読み込みフロー（もう少し細かく）

1. **CLI解析**

   * `CliArgs` 取得
2. **パス解決**

   * `config::resolve_pkg_path(&cli)` → `ResolvedConfig { pkg_path }`
3. **help.pkg 展開**

   * `extract::prepare_pkg(&pkg_path)` → `ExtractedPkg { root_dir }`
4. **メタ情報読み込み**

   * `pkg::meta::load_meta(&root_dir)` → `Meta`
5. **トピック読み込み**

   * `pkg::topics::load_topics(&root_dir)` → `Topics`
6. **AppState 構築**

   * `AppState { extracted, meta, topics, initial_topic: cli.topic.clone() }`
7. **WebView起動**

   * `webview::run_event_loop(state)`

---

## 6. ここまでで決まっていること・まだ余地があるところ

### 確定寄りな部分

* プロジェクトは単一バイナリ crate（`help-viewer`）
* モジュール単位：`cli`, `pkg`, `extract`, `app` などに分割
* 起動時のフロー（CLI→help.pkg 展開→JSON読込→WebView起動）
* `topics.json` による topic ID → ページURL 解決

### まだ設計余地のある部分

* 展開先キャッシュ戦略（毎回展開 / ハッシュによる再利用）
* `search_index.json` の扱い（JS側だけか、Rust 側も関与するか）
* TOC を Rust 側で使うか（とりあえずは JS 側に任せるのもあり）
* 多言語・複数 help.pkg の切替（将来的な課題）
