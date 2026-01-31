# gdrive-tree 仕様メモ（たたき台）

## 目的
- Google Drive の内容を Notion 風の UI で閲覧・編集できる Web アプリを作る。
- 左ペインにファイルツリー、右ペインにコンテンツ表示（埋め込み / 編集）を行う。
- Vue フレームワークで実装。
- 静的ビルドで公開（SPA）。

## MVP 要件
1. Google 認証（OAuth 2.0 + PKCE）
2. 左ペインに Drive のファイルツリー表示
3. 右ペインに選択ファイルの内容を表示
4. ファイル選択で素早く切り替え

## 対象ファイル種別（初期）
- Google ドキュメント / スプレッドシート / スライド
- PDF / 画像 / テキスト
- フォルダ

## UI/UX
- 2 ペインレイアウト（左: ツリー / 右: コンテンツ）
- 左ペインはフォルダ展開・折りたたみ
- 右ペインは選択ファイルを即時表示
- 検索バー（ファイル名）
- 最近開いた一覧（履歴）

## 技術スタック案
- フロント: Vue 3 + Vite + TypeScript
- 認証: Google Identity Services（OAuth 2.0）
- API: Google Drive API v3
- ホスティング: 静的ホスティング（GitHub Pages / Cloudflare Pages / Vercel）

## Google API 設計
### OAuth
- スコープ（最小）
  - https://www.googleapis.com/auth/drive.readonly
  - 編集が必要なら https://www.googleapis.com/auth/drive
- アクセストークンはメモリ保持（可能なら）
- リフレッシュトークンは原則使わず、再認証で更新
- Google Identity Services（OAuth 2.0 Token Client）を使用
- 環境変数: `VITE_GOOGLE_CLIENT_ID`
- 許可する JavaScript オリジンに `http://localhost:5173` を登録

### Drive API v3
- ファイル一覧: `files.list`
  - fields: `files(id,name,mimeType,parents,modifiedTime,iconLink,webViewLink,webContentLink)`
  - supportsAllDrives: true（共有ドライブ対応）
  - corpora: user / drive
- フォルダ判定: mimeType = `application/vnd.google-apps.folder`

## ツリー構築方針
- ルート配下から階層的に取得
- 初回はルート配下だけ取得し、展開時に子フォルダを遅延ロード
- 取得済みノードはキャッシュ
- ノード構造: { id, name, mimeType, parentId, children?, loaded? }

## 右ペイン表示（iframe 方式）
### 目的
- Google Docs/Sheets のフル機能 UI をそのまま使う。

### 埋め込み方法
- Drive の `webViewLink` を iframe に指定
- もしくは `https://drive.google.com/file/d/{fileId}/preview`
- Google Docs/Sheets/Slides は `.../preview` でビュー表示可能

### 制約
- 一部のファイルタイプや設定で `X-Frame-Options` により埋め込み不可の場合あり
- フル編集 UI（docs.google.com の編集画面）が iframe で禁止される可能性
- CSP / サードパーティ Cookie 制限で表示崩れが起きる場合あり
- 編集 iframe は試せるが、原則ブロックされる前提で設計する

## iframe 以外の選択肢
1. **新しいタブで `webViewLink` を開く**
   - 最も確実にフル機能を使える
   - 右ペイン内表示ではない
2. **Google Docs API / Sheets API で独自描画**
   - 表示・編集をアプリ内に組み込めるが、Docs の完全互換は難しい
3. **Google Drive リッチプレビュー（preview）**
   - 表示は可能だが編集機能は制限される

## パフォーマンス
- フォルダ展開時の遅延ロード
- ファイル切り替え時は iframe の src 差し替えで即時表示
- API レスポンスのキャッシュ（メモリ）

## セキュリティ
- トークンは本来メモリ保持を推奨だが、ログイン維持のため localStorage で短期キャッシュを許容
- CSP で許可するフレームソースを限定
- 開発・本番で OAuth クライアントを分ける

## 静的ビルド/公開
- Vite の `build` で静的生成
- SPA ルーティングはリライト設定が必要（GitHub Pages なら hash router）

## 開発手順（Deno）
- 開発サーバ: `deno task dev`
- ビルド: `deno task build`
- プレビュー: `deno task preview`
- 依存関係は Deno の npm 互換で取得

## 未確定事項（要決定）
- 対象ファイルタイプの範囲
- 共有ドライブの扱い
- 編集を iframe 内でどこまで許可するか
- 認証の再ログイン UX
- 右ペインの表示形式（iframe / 外部タブ / 独自ビュー）

## 次のアクション
- 仕様の優先順位を決める
- iframe で「編集画面が可能か」実機検証
- MVP の画面モック作成
- Vue プロジェクト作成と Google OAuth 設定
- Drive API 取得のページング対応
