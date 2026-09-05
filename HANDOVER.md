# MD Viewer PWA ── 引き継ぎ資料

**作成日**: 2026-03-17
**対象**: 新しいタスク（Claudeセッション）に引き継ぐための現状まとめ

---

## びばの全プロジェクト

| プロジェクト | パス | 地図 |
|-------------|------|------|
| **TradeBot** | `/Users/Vaio/Project/01_TradeBot/` | `_map/` |
| **note執筆** | `~/マイドライブ/2001_Project/article/` | `_map/` |
| **自主勉強** | `~/マイドライブ/1000_jishu-benkyo/` | `_map/` |
| **knowledge-base** | `~/マイドライブ/2001_Project/knowledge_base/` | `_map/` |
| **2001_Project全体** | `~/マイドライブ/2001_Project/` | `_map/` |
| **md-viewer-pwa** | `~/マイドライブ/2001_Project/md-viewer-pwa/` | **ここ** |

---

## 1. プロジェクト概要

Google Drive の `.md` ファイルを iPhone/iPad で閲覧・編集し、GitHub に push できる PWA。
単一の `index.html`（約1,370行）＋ `sw.js` ＋ `manifest.json` の3ファイル構成。

### ファイル構成

```
md-viewer-pwa/
├── index.html      … アプリ本体（HTML + CSS + JS を1ファイルに集約）
├── manifest.json   … PWA マニフェスト
├── sw.js           … Service Worker（オフラインキャッシュ）
├── setup-guide.md  … 初回セットアップ手順書
└── HANDOVER.md     … この引き継ぎ資料
```

---

## 2. 前回セッションで実装済みの内容

以下4点を `index.html` に反映済み。

### (a) Client ID をデフォルト値として埋め込み

- **場所**: `index.html` 551行目付近
- **内容**: `<input>` の `value` 属性に Client ID をハードコード
- **理由**: びば専用アプリなので毎回入力する必要をなくした
- **リセット時**: `clearAllSettings()` で同じデフォルト値に戻る（1316行目付近）

```html
<input type="text" id="gdrive-client-id"
  value="975677173358-xxxxxxxx.apps.googleusercontent.com"
  style="font-size:12px;color:var(--text-muted);">
```

### (b) GitHub PAT 欄に autocomplete 属性追加（Chrome パスワード保存対応）

- **場所**: `index.html` 566〜569行目付近
- **内容**: `<form>` タグで囲み、`autocomplete="on"` を設定。隠し `username` フィールド＋ `type="password"` ＋ `autocomplete="current-password"` の組み合わせで Chrome のパスワード保存ダイアログを発火させる
- **理由**: PAT はセキュリティ上 localStorage に保存したくないが、毎回手入力は面倒 → Chrome のパスワードマネージャに委任

```html
<form id="github-pat-form" onsubmit="event.preventDefault(); fetchGithubRepos();" autocomplete="on">
  <input type="hidden" name="username" autocomplete="username" value="md-viewer-pwa">
  <input type="password" id="github-token" name="password" autocomplete="current-password" placeholder="ghp_xxxxxxxxxxxx">
</form>
```

### (c) localStorage で非秘密設定を保存・復元

- **場所**: `index.html` 622〜676行目付近
- **保存対象**（キー: `mdviewer_settings`）:
  - `gdriveClientId` … Google Drive Client ID
  - `gdriveFolderId` … Google Drive フォルダID
  - `githubRepo` … リポジトリ名（owner/repo）
  - `githubBranch` … ブランチ名
  - `githubPath` … 保存先パス
- **保存しないもの**: `gdriveToken`（OAuth）、`githubToken`（PAT） → メモリのみ
- **復元タイミング**: ページ読み込み時に即時実行関数 `restoreSettings()` で各フォーム欄に反映

### (d) データ管理セクションの説明文を更新

- **場所**: `index.html` 593〜595行目付近
- **内容**: 「リポジトリ名やブランチなどの設定はブラウザに保存されます。トークンは Chrome のパスワード保存機能で管理できます。」という説明文に更新済み

---

## 3. アプリの状態管理（設計メモ）

```
┌─────────────────────────────────────────────────┐
│  state オブジェクト（JavaScript メモリ上）        │
│                                                   │
│  ◆ 保存する（「この端末に保存」が既定ON）         │
│    - gdriveToken       … OAuth アクセストークン    │
│    - gdriveTokenExpiry … 失効時刻(ms)             │
│    - gdriveConnected   … 接続実績フラグ            │
│    - githubToken       … GitHub PAT               │
│  ◆ メモリのみ（ページ閉じたら消える）             │
│    - files[]        … ファイル一覧                │
│    - currentFile    … 現在表示中のファイル         │
│    - currentContent … Markdown ソース             │
│                                                   │
│  ◆ localStorage に永続化                          │
│    - gdriveClientId … Client ID                   │
│    - gdriveFolderId … フォルダID                  │
│    - githubRepo     … リポジトリ名                │
│    - githubBranch   … ブランチ                    │
│    - githubPath     … 保存先パス                  │
└─────────────────────────────────────────────────┘
```

---

## 4. 使っている外部ライブラリ

| ライブラリ | 用途 | CDN 読み込み |
|---|---|---|
| marked | Markdown → HTML 変換 | jsdelivr |
| highlight.js | コードのシンタックスハイライト | jsdelivr |
| mermaid | Mermaid 図表レンダリング | jsdelivr |
| KaTeX | LaTeX 数式レンダリング | jsdelivr |
| Google Identity Services | Google OAuth2 認証 | googleapis |

---

## 4.5 HTML プレビュー（ブラウザ同等表示）

`.html` / `.htm` / `.xhtml` は **サンドボックス iframe でレンダリング表示**する（従来はシンタックスハイライト付きのソース表示のみだった）。

| 項目 | 内容 |
|---|---|
| ファイルタイプ | `getFileType()` が `'html'` を返す（`'code'` より先に判定） |
| 描画 | `renderHtmlPreview()` → `htmlPreviewInline()` → `mountHtmlPreview()` |
| 表示切替 | ヘッダーの「コード」/「プレビュー」ボタン（CSV の「ソース」/「テーブル」と同じ仕組み） |
| ツールバー | ◀ 戻る / ▶ 進む / ⟳ 再読込 / ⛶ 全画面 |

> ツールバーにファイル名は出さない。ビューアヘッダーとタブに既に出ていて三重になるため。
> 未解決の参照があるときだけ件数を表示する。
> 「別タブで開く」も置かない。blob URL で開くと相対リンクが解決できず、
> ページを行き来できないので教科書用途では役に立たない。
| 枠の高さ | `sizeHtmlPreview()` が実測して画面に収める（ツールバーが sticky ヘッダーに隠れないため） |

### 相対パス参照の解決

HTML 内の相対参照（`./style.css`、`img/logo.png` など）は、`state.files` の `drivePath` を
突き合わせて Drive 上の実体を取得し、単体で完結する HTML に組み立て直してから流し込む。

- `<link rel=stylesheet>` → `<style>` に展開（内部の `url()` / `@import` も再帰的に解決）
- `<script src>` → 中身をインライン化
- `img` / `source` / `video` / `audio` / `embed` / `object` / `srcset` → data URI 化
- `<style>` ブロック・`style` 属性内の `url()` → data URI 化
- 絶対URL（`https://` / `data:` / `//`）はそのまま残す → CDN 参照は通常どおり動く
- `<base href>` があるページは作者の指定を尊重して解決しない
- 見つからなかった参照はツールバーに「未解決 N 件」と表示（`title` に一覧）

### ページ間リンクの遷移

iframe に注入したシムが相対リンクのクリックを横取りし、`postMessage` で親アプリに
`{ __folioNav: true, href }` を送る。親は `htmlPreviewNavigate()` で Drive 上のファイルに
解決し、**同じタブのまま次のページを描画**する（タブの対象ファイルも差し替わるので、
編集・保存・Push は「いま見ているページ」に対して働く）。

- `ch1.html` / `chapters/ch2.html` / `../index.html` などの相対パスに対応
- 拡張子なし（`chapter1`）やディレクトリ指定（`chapters/`）は
  `.html` → `.htm` → `/index.html` → `/index.htm` の順で候補を試す
- `page.html#sec2` はページ遷移後にそのアンカーへスクロール
- ページ内アンカー（`#id`）は srcdoc だと既定動作が効かないのでシム側で `scrollIntoView`
- 絶対URL は横取りせず iframe 内でそのまま遷移
- HTML 以外（PDF/画像/md）へのリンクは `openFile()` で通常のファイルとして開く
- 解決できないリンクはトーストで通知して遷移しない
- 戻る/進む履歴はタブごと（`tab.navBack` / `tab.navFwd`）に保持

親側の `message` リスナーは `ev.source !== frame.contentWindow` を弾く。
sandbox に `allow-same-origin` がないので origin は `"null"` になり、origin 比較は使えない。

### セキュリティ

`sandbox="allow-scripts allow-popups allow-popups-to-escape-sandbox allow-modals allow-forms allow-downloads"`。
**`allow-same-origin` は付けない** ため、プレビュー内のスクリプトから親アプリの DOM や
Drive / GitHub のトークンには一切到達できない（`parent.document` は `SecurityError`）。

その副作用で iframe 内の `localStorage` / `sessionStorage` 参照が例外を投げるので、
`HTML_PREVIEW_SHIM` をページ先頭に注入してメモリ実装に差し替えている（既存ページが落ちないようにするため）。

### タブ状態

iframe は `innerHTML` で復元すると再実行されて壊れるため、PDF / 画像と同様に
`tab.renderedHtml` のキャッシュ対象から除外し、タブ復帰時は `content` から再描画する。

---

## 4.6 起動時の自動接続（認証の再現）

「前回開いたときの状態をそのまま復元する」方針。ページを開いたら無操作で Drive / GitHub に繋がる。

### GitHub

PAT とマッピングを localStorage に保存し、起動時に復元してバッジも「接続済み」に戻す。
PAT に有効期限がなければこれで恒久的に自動。「この端末に保存」チェックは**既定 ON**
（以前は既定 OFF だったため、毎回入力し直しになっていた）。

### Google Drive

アクセストークンは 1 時間で切れ、implicit flow にリフレッシュトークンが無いため段階的に試す。
`bootstrapDriveAuth()` が起動時に以下を順に実行する。

| 段階 | 手段 | 備考 |
|---|---|---|
| 1 | 保存トークンをそのまま使う | `gdriveTokenExpiry` が期限内のときだけ |
| 2 | hidden iframe + `prompt=none` | `silentRefreshDriveToken()`。サードパーティ Cookie が塞がれていると失敗する |
| 3 | **トップレベル遷移 + `prompt=none`** | `tryAutoRedirectAuth()`。一次パーティ Cookie が使えるので通る |
| 4 | 手動接続の案内 | Google 自体からログアウトしている場合はここに落ちる |

- 有効期限は `expires_in` から **5 分早める**（期限ギリギリのリクエスト失敗を防ぐ）
- `visibilitychange` で復帰時に期限切れなら黙って取り直す
- 手動接続の `prompt=consent` を**削除**した。これが「毎回同意画面が出る」原因。
  省略すれば初回だけ同意を求め、以降は無操作で戻る

> **注意**: 段階 2 が失敗するのは Safari だけの話ではない。iOS は Chrome を含む
> すべてのブラウザが WebKit を使うため、iPhone では**ブラウザを問わず**同じ制限を受ける。
> 「Chrome だから段階 3 は不要」と判断して消さないこと。

### リダイレクトのループ防止

段階 3 は画面が Google に飛ぶので、失敗したときに無限ループしないよう
`sessionStorage['mdviewer_auto_auth_tried']` で **1 セッション 1 回**に制限する。
`#error=` で戻ってきたときはガードを残したまま手動接続を案内し、
`#access_token=` で成功したときだけガードを解除する。

### 注意: `gapiReady` は `var` で宣言する

`gapiReady` の宣言は起動時の復元処理より**後**の行にある。`let` だと TDZ で
`ReferenceError` になり、**それ以降のスクリプトが丸ごと停止する**。
トークン保存が既定 OFF だった間はこの経路に入らず表面化していなかったが、
既定 ON にしたことで顕在化したため `var` に変更した。

---

## 4.7 デザイン（管理画面テンプレート風）

配色・余白・タイポを Craft のデザイン言語に寄せて全面的に作り直した。

### 方針

参考にしたのは Codebase 系の管理画面テンプレート。

- **淡いグレーの地に白い面**。グラデーションの発光オーブは削除
- 角丸はほぼ無し（`--radius: 5px`）、影も最小限
- 面の中は**ヘアラインで仕切って隙間なく**並べる（`.file-list` が枠、行が `border-bottom`）
- 種別アイコンは塗りバッジをやめ、**1 種類 1 色の線画**（`FILE_GLYPHS`）
- アクセントは青。状態色は緑 / 赤

### タイルの列数とサムネの大きさ

列数は **2 → 3 → 4 で頭打ち**（`620px` と `900px` のブレークポイント）。
`auto-fill` を使うと画面が広いほど列が増え、1 枚あたりが小さくなるので上限を切っている。

- サムネ枠は `width:100%` + `aspect-ratio: 1/1`（正方形）。`max-height` は掛けない
  （掛けると正方形が崩れる）
- 画像は **`object-fit: cover` + `object-position: top center`**。
  `contain` だと A4（1:1.41 の縦長）が枠の高さに合わせて縮み、左右が余白だらけで
  小さく見える。上端基準で切るので表紙や見出しは残る
- サムネが無いものの線画アイコンは `clamp(44px, 40%, 110px)`。固定 px だと
  大きな枠の真ん中に小さく浮いて間延びする

**実測（サムネの表示サイズ）**

| 端末幅 | 列数 | サムネ |
|---|---|---|
| iPhone SE 375px | 2 | 148×148 |
| iPhone 430px | 2 | 176×176 |
| iPad 縦 820px | 3 | 239×239 |
| iPad 横 1180px | 4 | 264×264 |
- 列数が可変なので `nth-child` で最終列の右線を消せない。
  右線は全部に引き、`.file-list` の `overflow:hidden` ではみ出しを切っている

### 表示形式（リスト / タイル）

`#file-list` に `.view-tile` を付けるかどうかで切り替える。選択は
`localStorage['mdviewer_view']` に保存し、次回起動時も維持する。

`renderFileList()` は途中 return が多いため、実描画は `renderFileListInner()` に分け、
外側の `renderFileList()` が `applyViewMode()` と `hydrateThumbs()` を必ず通すようにしている。
**新しく早期 return を足しても後処理が飛ばない**のはこの構造のおかげ。

### サムネイル（タイル表示時のみ）

PDF と画像は Drive の `thumbnailLink` を使う。実体をダウンロードするより遥かに軽い。

**3 段構えで取得する**（`buildThumbUrl`）。上から順に試し、取れた時点で終わる。

| 段 | 手段 | 備考 |
|---|---|---|
| 1 | `thumbnailLink` を認証付き fetch | 最も軽い。ただし googleusercontent 側が CORS を返さないことがあり、失敗は想定内 |
| 2 | 画像は実体を取って blob 化 | ビューアと同じ経路なので確実に通る |
| 3 | PDF は pdf.js で 1 ページ目を描画 | canvas → JPEG blob。API の使い方は `renderPdf` と同じ |

- 実体から作る場合は `THUMB_MAX_BYTES`（12MB）を超えたら諦める
- `thumbCache` は **Promise** を保持する。同じファイルの二重取得を防ぐため
- 一覧取得の `fields` に `thumbnailLink` を追加してある

> **注意**: 取得する `fields` を変えたら `FILE_CACHE_KEY` の版を上げること。
> 上げないと古い形のキャッシュを 24 時間掴み続ける。実際、`thumbnailLink` を
> 追加した際に版を上げ忘れ、サムネが一切出ない状態になった。
- `IntersectionObserver` で画面に入ったものから読む。`thumbCache` に blob URL を保持し、
  失敗は `null` を入れて再試行しない（線画アイコンのまま）
- `thumbnailLink` には有効期限があるため、localStorage のファイルキャッシュから
  復元した直後は失効していることがある。その場合は線画アイコンにフォールバックする

> **注意**: サムネの `<img>` に `loading="lazy"` を付けてはいけない。
> DOM に入れる前に読み込ませているため、付けると読み込みが始まらず `onload` が
> 永久に来ない（遅延読み込みは IntersectionObserver 側で行っている）。

> **注意**: 一覧の行を作る箇所は `renderFileList()` 内に **2 つある**
> （検索・フラット表示用と、フォルダ内表示用）。変数名が `iconClass` と `iconCls` で
> 違うため片方だけ直しやすい。アイコンを変えるときは必ず両方直すこと。

### 配色トークン

**色は必ず `:root` のトークン経由で参照する。個別ルールに生の色を書かない。**
以前は `rgba(10,14,26,.6)` のようなダーク前提の値が 60 箇所ほど散らばっていて、
ライト対応の妨げになっていた。現在、生の `rgba()` はトークン定義内にしか存在しない。

主なトークン: `--bg` `--card-bg` `--bg-hover` `--bg-sunken` `--overlay-soft` `--nav-bg`
`--text` `--text-secondary` `--text-muted` `--accent` `--accent-soft` `--accent-glow`
`--border` `--border-strong` `--success-soft` `--danger-soft` `--shadow-sm/md/lg`

### ライト / ダーク

| 指定 | 効き方 |
|---|---|
| `auto`（既定） | OS の `prefers-color-scheme` に追従。OS 側の変更にもその場で反応する |
| `light` / `dark` | `<html data-theme="...">` を付けて OS 設定より優先 |

- 設定画面の「外観」から切替。保存キーは `mdviewer_theme`（**設定本体とは別**。
  「設定をクリア」で外観まで戻さないため）
- 切替時に `<meta name="theme-color">` も実際の背景色に更新する（iOS のステータスバー対策）
- ダークは黒紺ではなく**温かいグレー**（`#1a1a19` 系）にして、ライトと地続きにしている

### アイコン

- フォルダは絵文字 `📁` をやめて SVG（黄色い絵文字が青系パレットから浮いていた）
- ファイル種別アイコンはグラデーションをやめて**単色の小さな差し色**に

---

## 4.8 スクロールの仕組み（アドレスバーを畳ませる）

既定の `html, body` は `height:100%` + `overflow:hidden` で、**中の箱（`.page`）だけが動く**
アプリ風レイアウト。この状態では Safari は「ページはスクロールしていない」と見なすため、
**アドレスバーを畳んでくれず、`position:sticky` も効かない**。

一覧とビューアでは `<html>` に `scroll-mode` を付けて document 自体を伸ばす。

```css
html.scroll-mode, html.scroll-mode body {
  height: auto; min-height: 100%; overflow: visible;
}
```

> **注意（実際に踏んだ順）**
> 1. `body` にだけ `overflow-y:auto` を付けても駄目。`html` の `overflow:hidden` が
>    残って document が動かない。**クラスは `<html>` にも付けること**
> 2. `body` の `height:100%` を残すと body 自身が内部スクロールし、document は伸びない
> 3. `overflow-y` だけ変えると `overflow-x:hidden` が残り、body が
>    スクロールコンテナ扱いになって `sticky` が効かない。`overflow: visible` にすること

`body.files-mode` / `body.viewer-mode` は、それ以外（`#app` の高さ、`main` の overflow、
sticky ヘッダー、`nav` の固定）に使う。

### 一覧のスクロール位置の復元

`filesScrollY` に覚えて、戻ったときに復元する。

> **注意**: 保存は `showPage()` の**先頭**で行うこと。ページを `display:none` にした後だと
> document が縮んで `window.scrollY` が 0 になり、必ず 0 が保存される（実際に踏んだ）。
> 復元は `requestAnimationFrame` を 2 回重ねる。1 フレームではレイアウトが間に合わない。

### ビューアの上部を詰めてある

ビューアでは `header`（アプリ名 + 同期ボタン）を `display:none` にしている。
戻るボタンとファイル名は下の行にあり、同期ボタンは一覧でしか使わないため、
読書中は 46px まるごと無駄だった。`.viewer-sticky-header` の `top` も 0 にしてある
（header を出す場合は `top` を戻すこと）。

---

## 5. 未着手・今後の検討事項

> 以下は前回セッション時点で話題に上がっていた、または改善余地のある項目。

1. **GitHub Pages へのデプロイ**: ファイルは完成しているがリポジトリへの push / Pages 設定はまだ（setup-guide.md に手順あり）
2. **PWA アイコン**: 現在は SVG のインラインアイコンのみ。実機テスト後に PNG アイコンが必要になる可能性あり
3. **共有端末での利用**: トークンを localStorage に保存するため、共有端末では「この端末に保存」を OFF にする
4. **オフライン時の編集同期**: 現状オフラインキャッシュは閲覧のみ。オフラインで編集 → オンライン復帰時に push するキュー機構は未実装
5. **Git コミット管理**: リポジトリに `.git` はあるが、まだコミット履歴なし（`git log` で確認済み）

---

## 6. 新しいセッションへの申し送り

新しいタスクを開いたら、以下をコピペして使ってください。

```
# コンテキスト
md-viewer-pwa/ フォルダに PWA アプリがある。
HANDOVER.md に前回までの作業内容をまとめてある。
まず HANDOVER.md を読んで現状を把握してほしい。

# 今回やりたいこと
（ここに次のタスクを書く）
```

---

## 要確認

- Google Cloud Console の OAuth 同意画面は「テスト」ステータスのまま？ → 本番公開（自分だけ使うなら不要だが、テストユーザー上限に注意）
- GitHub PAT の有効期限はいつまで設定した？ → 期限切れ前にリマインド手段を検討
