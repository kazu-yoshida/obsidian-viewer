# Google Client ID / API Key 公開問題の修正

## 問題

`index.html` に Google OAuth Client ID と Google API Key が直接ハードコードされており、
このリポジトリが public なため、誰でもソースコードから値を取得できる状態だった。

```js
const CLIENT_ID = '455215896385-8ae1qtobhhcf7pmm1g968udpkigat6kl.apps.googleusercontent.com';
const API_KEY   = 'AIzaSyBtnRIbNn-...（削除済み）';
```

## 対応内容

### 1. コード側の変更

- `index.html` の `CLIENT_ID` / `API_KEY` を実値からプレースホルダーに変更した。

  ```js
  const CLIENT_ID = '__GOOGLE_CLIENT_ID__';
  const API_KEY   = '__GOOGLE_API_KEY__';
  ```

- `.github/workflows/deploy.yml` を追加し、GitHub Pages へのデプロイ時に GitHub リポジトリ
  シークレットの値でプレースホルダーを置換してから公開するようにした。

  ```yaml
  - name: Inject secrets into index.html
    env:
      GOOGLE_CLIENT_ID: ${{ secrets.GOOGLE_CLIENT_ID }}
      GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
    run: |
      mkdir -p dist
      cp index.html dist/index.html
      sed -i "s|__GOOGLE_CLIENT_ID__|${GOOGLE_CLIENT_ID}|g" dist/index.html
      sed -i "s|__GOOGLE_API_KEY__|${GOOGLE_API_KEY}|g" dist/index.html
  ```

  これにより、リポジトリ上のソースコード（`main` ブランチの `index.html`）には実際の値が
  一切残らない。

- Client ID は仕組み上公開されても問題ない値（OAuth の性質上、ブラウザに必ず露出する）
  なので、**同じ値をそのまま**シークレットに登録した。API Key は不特定多数のクォータ消費・
  悪用リスクがあるため、**新しいキーを再発行**して差し替えた。

### 2. GitHub 側の設定

- Settings → Secrets and variables → Actions に以下を登録:
  - `GOOGLE_CLIENT_ID`（既存の値をそのまま）
  - `GOOGLE_API_KEY`（新しく発行したキー）
- Settings → Pages → Source を「Deploy from a branch」から **「GitHub Actions」** に変更。

### 3. Google Cloud Console 側の設定

- 漏洩していた旧 API Key（`AIzaSyBtnRIbNn-...`）を削除。
- 新しい API Key を発行し、以下の制限をかけた:
  - **アプリケーションの制限**: ウェブサイト（HTTP リファラー）
    - `https://kazu-yoshida.github.io/*`
    - `https://kazu-yoshida.github.io/obsidian-viewer/*`
  - **API の制限**: Google Drive API のみ許可

## 発生したトラブルと解決

| 症状 | 原因 | 対応 |
|---|---|---|
| Actions が `cp: cannot copy a directory, '.', into itself` で失敗 | `cp -r . dist/` が `dist` 自身を再帰的にコピーしようとしていた | `cp index.html dist/index.html` に変更し、必要なファイルだけを明示的にコピーするよう修正 |
| Secret を更新してもサイトの値が古いまま | Secret はワークフロー実行時にのみ読み込まれるため、更新後に再実行していなかった | Actions タブから該当ワークフローを **Re-run all jobs** で再実行 |
| ログイン後「フォルダの読み込みに失敗しました: Cannot read properties of undefined (reading 'files')」 | Drive API の discovery ドキュメント取得が `403 API_KEY_HTTP_REFERRER_BLOCKED` で拒否されていた（登録したリファラーとブラウザが実際に送る値が不一致） | API Key のウェブサイト制限に `https://kazu-yoshida.github.io/*` と `.../obsidian-viewer/*` の両方を登録 |

## 今後の注意点

- `index.html` 以外に CSS・画像・JS などのファイルを追加する場合は、
  `.github/workflows/deploy.yml` の `cp index.html dist/index.html` の行に
  そのファイル／フォルダのコピーも追記する必要がある。
- API Key や Client ID を変更する場合は、GitHub の Secrets 更新後に
  Actions ワークフローを手動で再実行（Re-run）しないと反映されない。
- ローカルで動作確認したい場合は、`index.html` の2行（`CLIENT_ID` / `API_KEY`）を
  一時的に実値に書き換えて確認し、**コミットしないこと**。
