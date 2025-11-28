# CSS

## 1. 全体レイアウトの基本

### コンテナと本文

```css
body {
  margin: 0;
  padding: 0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    sans-serif;
  font-size: 14px;
  line-height: 1.6;
  color: #222;
  background-color: #fff;
}

.hv-header {
  position: sticky;
  top: 0;
  z-index: 100; /* 結果パネルが本文より上に乗るように */
  padding: 6px 8px;
  background-color: #f5f5f5;
  border-bottom: 1px solid #ddd;
}

.hv-main {
  display: block;
  padding: 12px 16px;
}

.hv-content {
  max-width: 960px;
  margin: 0 auto;
}
```

* `position: sticky` にしてヘッダ（検索欄）をスクロールに追従。
* フォントはシステムUI系にして、追加フォントを入れない＝軽量。

---

## 2. 検索ボックスの見た目

```css
#hv-search-input {
  box-sizing: border-box;
  width: 100%;
  max-width: 420px;
  padding: 6px 8px;
  border-radius: 4px;
  border: 1px solid #bbb;
  font-size: 13px;
  outline: none;
  background-color: #fff;
}

#hv-search-input:focus {
  border-color: #4a8af4;
  box-shadow: 0 0 0 1px rgba(74, 138, 244, 0.15);
}
```

* 幅はヘッダ内で最大 420px くらい。ウィンドウが狭い時は100%。
* フォーカス時だけ軽く色をつけて「ちゃんと反応しているのがわかる」程度の装飾。

---

## 3. 検索結果パネル（ドロップダウン）

### ベース

```css
.hv-search-results {
  position: absolute;
  margin-top: 4px;
  /* headerのパディングと合わせる */
  left: 8px;
  right: 8px;
  max-width: 600px;

  /* デフォルト非表示（JSで切り替え） */
  display: none;

  background-color: #fff;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow:
    0 2px 4px rgba(0, 0, 0, 0.08),
    0 0 0 1px rgba(0, 0, 0, 0.02);

  max-height: 320px;
  overflow-y: auto;
  padding: 4px 0;
  z-index: 200;
}

/* JS側で .hv-search-results--visible を付けたときに表示 */
.hv-search-results.hv-search-results--visible {
  display: block;
}
```

ポイント：

* `position: absolute` + 左右固定で、検索欄のすぐ下にパネルを出す。
* `max-height: 320px` で縦に伸びすぎないようにして、`overflow-y: auto` でスクロール。
* box-shadow は1段＋枠1px程度にして、あまり重たくしない。

---

## 4. 検索結果項目のスタイル

```css
.hv-result {
  padding: 6px 10px;
  cursor: pointer;
  list-style: none;
}

.hv-result + .hv-result {
  border-top: 1px solid #eee;
}

.hv-result-title {
  font-size: 13px;
  font-weight: 600;
  color: #222;
  margin-bottom: 2px;
}

.hv-result-summary {
  font-size: 12px;
  color: #666;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;    /* WebKit系: 2行でトリミング */
  -webkit-box-orient: vertical;
}
```

* 行間は小さめにしてコンパクトに（でも読める程度）。
* `.hv-result + .hv-result` で項目間に薄い区切り線。
* `-webkit-line-clamp` は対応ブラウザが限定されますが、なくても単純に1行で `overflow: hidden` されるだけなので劣化に耐えられます。

### ホバーとアクティブ（キーボード選択）

```css
.hv-result:hover {
  background-color: #f3f6ff;
}

.hv-result--active {
  background-color: #e3ecff;
}
```

* `hover` と `active` で微妙に色を変えて、キーボード操作時とマウス操作時の違いがわかるようにしておく。

---

## 5. 「0件」メッセージのスタイル

```css
.hv-search-noresult {
  padding: 8px 10px;
  font-size: 12px;
  color: #777;
}
```

* 0件の時だけ `hv-search-results` の中身をこれ1行にして表示。
* 見た目は控えめに。

---

## 6. スクロール・高さの調整ポリシー

* 検索結果パネル：

  * `max-height: 320px`（だいたい6〜10件くらい見える高さ）
  * ウィンドウが極端に低い場合はブラウザ側が自動でスクロールをつけるので任せる。
* 本文エリア：

  * `.hv-main` のパディングで左・右・上下に 12〜16px 程度の余白。
  * これ以上凝ったレイアウト（2カラムなど）はヘルプ生成側に任せる。

---

## 7. 簡易ダークモード対応（オプション）

ダークモード対応するなら、`prefers-color-scheme` に乗るくらいの軽さで OK です。

```css
@media (prefers-color-scheme: dark) {
  body {
    background-color: #121212;
    color: #e0e0e0;
  }

  .hv-header {
    background-color: #1f1f1f;
    border-bottom-color: #333;
  }

  #hv-search-input {
    background-color: #181818;
    border-color: #444;
    color: #eee;
  }

  #hv-search-input:focus {
    border-color: #8ab4f8;
    box-shadow: 0 0 0 1px rgba(138, 180, 248, 0.3);
  }

  .hv-search-results {
    background-color: #232323;
    border-color: #444;
    box-shadow:
      0 2px 4px rgba(0, 0, 0, 0.7),
      0 0 0 1px rgba(255, 255, 255, 0.03);
  }

  .hv-result {
    border-top-color: #333;
  }

  .hv-result-title {
    color: #f5f5f5;
  }

  .hv-result-summary {
    color: #aaa;
  }

  .hv-result:hover {
    background-color: #273143;
  }

  .hv-result--active {
    background-color: #31415f;
  }

  .hv-search-noresult {
    color: #aaa;
  }
}
```

* ヘルプ側のCSSと衝突しにくいよう、`hv-` で名前空間をつくっているのがポイントです。

---

## 8. 依存を増やさないポリシー

デザイン方針として：

* CSSフレームワーク（Bootstrap など）は使わない（ファイルサイズ節約・競合回避）。
* フォントも外部CDNからは取らない（完全オフライン）。
* アニメーションはほぼ無し（結果パネルのフェードインくらい欲しくなったら後で追加）。
