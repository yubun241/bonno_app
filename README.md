# 煩悩ログ（GitHub Pages 版）

貪・瞋・痴のレベルと内容、四無量心（慈悲喜捨）のアクションプランを記録する単一ページアプリ。
データは **別のプライベートリポジトリ** の `bonno.json` に GitHub Contents API 経由で保存する。

## 構成

```
[iPhone Safari / ホーム画面PWA]
   │  index.html（このリポジトリ → GitHub Pages）
   │
   │  fetch https://api.github.com/repos/<owner>/<data-repo>/contents/bonno.json
   │  Authorization: Bearer <fine-grained PAT>
   ▼
[プライベートリポジトリ <data-repo>]  bonno.json
```

- アプリ側リポジトリ：秘密情報なし。Pages で配信される（URL を知れば誰でも開ける）
- データ側リポジトリ：プライベート。PAT はこのリポジトリの Contents 権限のみ
- 端末側：entries のキャッシュ・PAT・最後に見た blob sha を localStorage に保持。オフライン時はローカルに溜めて復帰時に同期

## セットアップ

### 1. データ用プライベートリポジトリ
1. 新規リポジトリ作成（例 `bonno-data`、**Private**、README 付きで初期化して `main` ブランチを作る）
2. ファイルは作らなくてよい（初回保存時にアプリが `bonno.json` を作成する）

### 2. Fine-grained PAT
GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate
- Repository access: **Only select repositories** → `bonno-data` のみ
- Permissions → Repository permissions → **Contents: Read and write**（Metadata は自動で Read）
- Expiration: 任意（切れたらアプリの設定で差し替え）

### 3. アプリ用リポジトリ → Pages
1. このフォルダの `index.html` と `README.md` をリポジトリ（例 `bonno-app`）に push
2. Settings → Pages → Build and deployment: **Deploy from a branch** → `main` / `/ (root)`
3. 数分後 `https://<owner>.github.io/bonno-app/` で開ける

> Pages をプライベートリポジトリから出すには GitHub Pro 以上が必要。Free の場合はアプリ側リポジトリを Public にする（中身は汎用 HTML のみ）。
> いずれの場合も Pages サイト自体は公開 URL になる（Private Pages は Enterprise のみ）。データはアプリ側に一切含まれない。

### 4. iPhone 側
1. Safari で Pages の URL を開く
2. 共有 → **ホーム画面に追加**（スタンドアロン表示・セーフエリア対応済み）
3. 最下部「GitHub 同期設定」に `owner/bonno-data`・`bonno.json`・`main`・PAT を入力 → 「保存して接続テスト」

## 同期の仕組み（設計メモ）

- **読み**：`GET /contents/{path}?ref={branch}` → base64 → JSON 配列。レスポンスの `sha` を保持
- **書き**：`PUT /contents/{path}` に `sha` を添えて更新。`409/422`（他端末が先に書いた）なら再 GET → マージ → 再 PUT
- **マージ**：`id` 単位で `updatedAt` が新しい方を採用。削除は `{deleted:true}` の tombstone として残し、他端末にも伝播させる
- **トリガ**：保存／削除時、アプリ復帰時（visibilitychange）、オンライン復帰時、手動「同期」ボタン
- 1ファイル全置換なので、数千件規模までは問題ない。それ以上なら月別ファイル分割に変える

## データ形式（`bonno.json`）

```json
[
  {
    "id": "m2x3k9ab1c",
    "ts": 1761444000000,
    "updatedAt": 1761444100000,
    "ton": { "level": 6, "attach": true, "attachText": "...", "desire": "..." },
    "jin": { "level": 3, "anger": "...", "sad": "..." },
    "chi": { "level": 4, "note": "..." },
    "action": { "ji": "...", "hi": "...", "ki": "...", "sha": "..." },
    "memo": "..."
  },
  { "id": "...", "ts": 1761000000000, "updatedAt": 1761500000000, "deleted": true }
]
```

## セキュリティ上の前提

- PAT は iPhone の Safari localStorage に平文で入る。端末ロックと PAT のスコープ最小化（1リポジトリ・Contents のみ・有効期限付き）で担保する
- Pages の URL は公開されるが、アプリには PAT もデータも含まれない。URL を知られてもデータは漏れない
- 端末紛失時は GitHub 側で PAT を revoke する
