# MD Viewer PWA ── 引き継ぎ資料

**作成日**: 2026-03-17
**対象**: 新しいタスク（Claudeセッション）に引き継ぐための現状まとめ

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
│  ◆ メモリのみ（ページ閉じたら消える）             │
│    - gdriveToken    … Google OAuth アクセストークン│
│    - githubToken    … GitHub PAT                  │
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

## 5. 未着手・今後の検討事項

> 以下は前回セッション時点で話題に上がっていた、または改善余地のある項目。

1. **GitHub Pages へのデプロイ**: ファイルは完成しているがリポジトリへの push / Pages 設定はまだ（setup-guide.md に手順あり）
2. **PWA アイコン**: 現在は SVG のインラインアイコンのみ。実機テスト後に PNG アイコンが必要になる可能性あり
3. **オフライン時の編集同期**: 現状オフラインキャッシュは閲覧のみ。オフラインで編集 → オンライン復帰時に push するキュー機構は未実装
4. **Git コミット管理**: リポジトリに `.git` はあるが、まだコミット履歴なし（`git log` で確認済み）

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
