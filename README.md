# incentive-board

インセンティブ獲得リアルタイム管理ボード（単一の `index.html` で動作する静的Webアプリ）。
スタッフごとのレシート登録・ランク別単価計算・月間ランキング・A4印刷用集計表を、
Firebase（Firestore / Authentication）を使って複数端末間でリアルタイムに同期します。

**Firebase Storageは使用していません。** レシート画像はブラウザ内でリサイズ・圧縮した上で
Firestoreのドキュメントに直接保存するため、クレジットカード登録が必須のBlazeプランへの
切り替えは不要で、無料のSparkプランのまま運用できます。

## 構成

- `index.html` — アプリ本体（UI + ロジック、Firebase SDKはCDNから読み込み）
- `firestore.rules` — Firestoreのセキュリティルール（Firebaseコンソールに貼り付けて使用）

## Firebase セットアップ手順

1. [Firebase コンソール](https://console.firebase.google.com/) で新しいプロジェクトを作成します。
2. **Firestore Database** を作成します（本番環境モードでOK。ルールは手順4で設定）。
3. **Authentication** で **匿名（Anonymous）** ログインを有効にします（Sign-in method タブ）。
   - ログイン画面なしでアプリを開いた人を自動的に匿名ユーザーとして認証し、
     セキュリティルールで「認証済みユーザーのみ読み書き可」を強制するための仕組みです。
4. Firestoreのルールを本リポジトリの `firestore.rules` の内容で公開します。
5. プロジェクトの設定 ＞「マイアプリ」でWebアプリを追加し、表示された設定オブジェクト（`firebaseConfig`）をコピーします。
   - この手順では **Storage は有効化しないでください**（有効化するとBlazeプランへの切り替えを求められます）。
6. `index.html` 内の以下の箇所に、コピーした値を貼り付けます。

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

7. `index.html` をブラウザで開く（またはGitHub Pagesなどにホスティングする）と、
   ヘッダー部分に同期状態（🔥 Firebase 連携中 / 💾 ローカルのみ動作中）が表示されます。

`firebaseConfig` を初期値（`YOUR_API_KEY` のまま）にしておくと、Firebaseに接続せず
このブラウザ内だけで完結するローカル動作モードとして使えます（データはリロードで消えます）。

## データ構造（Firestore コレクション）

- `rankMaster/{rankKey}` — `{ label, rate, order }` ランク名・単価・表示順
- `staff/{staffId}` — `{ name, rankKey }` スタッフ名と現在のランク
- `acquisitions/{acquisitionId}` — `{ staffId, staffName, type, time, month, image, createdAt }` 獲得実績1件ごとのレコード

`image` はレシート写真をリサイズ・JPEG圧縮したData URL文字列です。Firestoreの
1ドキュメントあたり上限（約1MB）に収まるよう、必要に応じて自動的に画質・解像度を落とします。

## 無料枠の使用量確認

「⚙️ スタッフ・マスタ管理」モーダル内の「Firestore無料枠 使用量確認」から、
現在の登録件数・画像の推定使用容量・無料枠（合計1GB）に対する残り保存可能枚数の目安を確認できます。
