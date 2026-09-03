# incentive-board

インセンティブ獲得リアルタイム管理ボード（単一の `index.html` で動作する静的Webアプリ）。
スタッフごとのレシート登録・ランク別単価計算・月間ランキング・A4印刷用集計表を、
Firebase（Firestore / Storage / Authentication）を使って複数端末間でリアルタイムに同期します。

## 構成

- `index.html` — アプリ本体（UI + ロジック、Firebase SDKはCDNから読み込み）
- `firestore.rules` — Firestoreのセキュリティルール（Firebaseコンソールに貼り付けて使用）
- `storage.rules` — Storageのセキュリティルール（Firebaseコンソールに貼り付けて使用）

## Firebase セットアップ手順

1. [Firebase コンソール](https://console.firebase.google.com/) で新しいプロジェクトを作成します。
2. **Firestore Database** を作成します（本番環境モードでOK。ルールは手順5で設定）。
3. **Storage** を有効化します（レシート画像の保存先）。
4. **Authentication** で **匿名（Anonymous）** ログインを有効にします（Sign-in method タブ）。
   - ログイン画面なしでアプリを開いた人を自動的に匿名ユーザーとして認証し、
     セキュリティルールで「認証済みユーザーのみ読み書き可」を強制するための仕組みです。
5. Firestore・Storageのルールをそれぞれ本リポジトリの `firestore.rules` / `storage.rules` の内容で公開します。
6. プロジェクトの設定 ＞「マイアプリ」でWebアプリを追加し、表示された設定オブジェクト（`firebaseConfig`）をコピーします。
7. `index.html` 内の以下の箇所に、コピーした値を貼り付けます。

   ```js
   const firebaseConfig = {
     apiKey: "YOUR_API_KEY",
     authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
     projectId: "YOUR_PROJECT_ID",
     storageBucket: "YOUR_PROJECT_ID.appspot.com",
     messagingSenderId: "YOUR_SENDER_ID",
     appId: "YOUR_APP_ID"
   };
   ```

8. `index.html` をブラウザで開く（またはGitHub Pagesなどにホスティングする）と、
   ヘッダー部分に同期状態（🔥 Firebase 連携中 / 💾 ローカルのみ動作中）が表示されます。

`firebaseConfig` を初期値（`YOUR_API_KEY` のまま）にしておくと、Firebaseに接続せず
このブラウザ内だけで完結するローカル動作モードとして使えます（データはリロードで消えます）。

## データ構造（Firestore コレクション）

- `rankMaster/{rankKey}` — `{ label, rate, order }` ランク名・単価・表示順
- `staff/{staffId}` — `{ name, rankKey }` スタッフ名と現在のランク
- `acquisitions/{acquisitionId}` — `{ staffId, staffName, type, time, month, imageUrl, createdAt }` 獲得実績1件ごとのレコード

レシート画像は Storage の `receipts/{month}/{timestamp}_{filename}` に保存され、
そのダウンロードURLが `acquisitions` ドキュメントの `imageUrl` に記録されます。
