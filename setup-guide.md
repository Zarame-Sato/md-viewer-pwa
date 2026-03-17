# MD Viewer PWA セットアップガイド

## この PWA でできること

- Google Drive に保存した `.md` ファイルを iPhone / iPad で美しく表示
- コードのシンタックスハイライト、Mermaid 図、LaTeX 数式に対応
- ファイルの編集（ソースモード）
- 編集した内容を GitHub に commit & push
- オフラインでも閲覧可能（Service Worker）

---

## セットアップ手順（3ステップ）

### ステップ1：PWA を GitHub Pages で公開する

**Why**: PWA は「Web サイト」なので、どこかにホスティングが必要。GitHub Pages なら無料。

1. GitHub で新しいリポジトリを作成（例：`md-viewer-pwa`）
2. 以下の3ファイルをリポジトリにアップロード
   - `index.html`
   - `manifest.json`
   - `sw.js`
3. リポジトリの Settings → Pages → Source を `main` ブランチに設定
4. 数分待つと `https://あなたのユーザー名.github.io/md-viewer-pwa/` でアクセス可能に

### ステップ2：Google Drive API を有効にする

**Why**: Google Drive のファイルを読み取るには Google の許可（OAuth2）が必要。

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. 新しいプロジェクトを作成（名前は何でもOK）
3. 「API とサービス」→「ライブラリ」→「Google Drive API」を有効化
4. 「API とサービス」→「認証情報」→「認証情報を作成」→「OAuth クライアント ID」
5. アプリケーションの種類：「ウェブ アプリケーション」
6. 承認済みの JavaScript 生成元に追加：
   ```
   https://あなたのユーザー名.github.io
   ```
7. 承認済みのリダイレクト URI に追加：
   ```
   https://あなたのユーザー名.github.io/md-viewer-pwa/index.html
   ```
8. 「OAuth 同意画面」を設定（テスト用なら「外部」→自分のメールだけテストユーザーに追加）
9. 発行された **Client ID**（`xxxxx.apps.googleusercontent.com`）をメモ

### ステップ3：GitHub Personal Access Token を発行する

**Why**: GitHub API でファイルを読み書きするための認証キー。

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 「Generate new token (classic)」をクリック
3. Note: `md-viewer-pwa`（何でもOK）
4. Expiration: お好みで（90 days が無難）
5. Scopes: `repo` にチェック
6. 「Generate token」→ 表示されたトークン `ghp_xxxx` をメモ
   - **このトークンは二度と表示されないのでコピー必須**

---

## iPad / iPhone にインストールする

1. Safari で `https://あなたのユーザー名.github.io/md-viewer-pwa/` を開く
2. 共有ボタン（□に↑のアイコン）をタップ
3. 「ホーム画面に追加」をタップ
4. 名前を「MD Viewer」にして「追加」

これでホーム画面にアプリアイコンが表示され、普通のアプリのように起動できます。

---

## アプリ内の設定

### Google Drive 接続

1. アプリの「設定」タブを開く
2. **Client ID** を入力（ステップ2で取得したもの）
3. **フォルダID**（任意）：
   - Google Drive でフォルダを開いたときの URL の末尾の文字列
   - 例: `https://drive.google.com/drive/folders/1ABCdef...` → `1ABCdef...`
   - 空欄にするとルートフォルダ以下の全 .md を検索
4. 「接続する」→ Google ログイン画面が表示される → 許可

### GitHub 接続

1. **Personal Access Token** を入力
2. **リポジトリ**: `owner/repo` 形式（例：`viva/my-notes`）
3. **ブランチ**: `main`（デフォルト）
4. **保存先パス**: リポジトリ内のフォルダ（例：`docs/`）空欄ならルート
5. 「接続テスト」→ 成功すれば「接続済み」と表示

---

## 使い方

### Markdown を見る

1. 「ファイル」タブでファイルをタップ
2. 美しく整形された Markdown が表示される

### 編集する

1. ファイル表示中に「編集」ボタンをタップ
2. 「ソース」タブで Markdown を直接編集
3. 「プレビュー」タブでリアルタイム確認
4. 「閲覧」ボタンで編集を終了

### GitHub に push する

1. ファイル表示中（または編集後）に「Push」ボタンをタップ
2. 自動でコミットメッセージが付いて push される

---

## セキュリティについて

- **トークンはメモリ内のみに保存**。ブラウザを閉じると消えるので、毎回入力が必要
- トークンを永続化したい場合は、PWA のコードを改修して暗号化保存を検討
- Personal Access Token は必ず最小限のスコープ（`repo` のみ）で発行
- 期限付きトークンを推奨（90日など）

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| Google 認証でポップアップが出ない | Safari の「ポップアップブロック」を解除 |
| Google 認証後にエラー | OAuth 同意画面でテストユーザーに自分を追加しているか確認 |
| GitHub push で 404 | リポジトリ名が `owner/repo` 形式か確認 |
| GitHub push で 409 | 同じファイルが別の場所から更新された → ブラウザを更新して再試行 |
| ファイルが表示されない | フォルダID が正しいか確認。空欄でテストしてみる |
