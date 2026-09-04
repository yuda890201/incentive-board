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
- `tasks/{taskId}` — `{ title, description, reward, status, claimedBy, claimedAt, submissionNote, submissionImage, submittedAt, approvedAt, approvedMonth, createdAt }` タスク掲示板の1件ごとのレコード。`status` は `open`（募集中）→`claimed`（対応中）→`submitted`（承認待ち）→`approved`（承認済み・報酬確定）と遷移します
- `competitions/{competitionId}` — `{ title, description, deadline, status, createdAt }` POP/SNSコンペの開催情報。`status` は `open` または `closed`
- `competitionEntries/{entryId}` — `{ competitionId, competitionTitle, staffId, staffName, entryType, image, link, comment, isWinner, prizeAmount, resultMonth, submittedAt }` コンペへの応募1件ごとのレコード。`entryType` が `image` の場合は圧縮画像、`link` の場合はSNS/DriveなどのURLを保存します

`image` はレシート写真・応募作品をリサイズ・JPEG圧縮したData URL文字列です。Firestoreの
1ドキュメントあたり上限（約1MB）に収まるよう、必要に応じて自動的に画質・解像度を落とします。

## タスク掲示板 / POP・SNSコンペ

留学生や社保の扶養に入っているスタッフなど、労働時間を抑えたいスタッフ向けに、
時給に依存しない成果報酬型のインセンティブを追加できる機能です（ヘッダーの「🎯 タスク＆コンペ」から利用）。

- **タスク掲示板**: マネージャーが低頻度タスク（POP作り直しなど）を報酬額付きで掲示 →
  スタッフが早い者勝ちで担当 → 完了報告（メモ＋任意で写真） → マネージャーが承認すると、
  その月の見込支給額に自動加算されます。
- **POP/SNSコンペ**: マネージャーがコンペを開催 → スタッフが作品を応募（画像は圧縮してFirestoreに保存、
  動画やSNS投稿はURLで応募） → マネージャー/オーナーが優勝作品を選び賞金額を入力すると、
  その月の見込支給額に自動加算されます。Storageを使わない構成上、動画ファイル自体のアップロードには対応していません。

ランキング表の「見込支給額」には、レシート実績による金額に加えて、承認済みタスクの報酬額・
コンペの獲得賞金額が自動的に合算されます（内訳は金額の下に小さく表示されます）。

## 無料枠の使用量確認

「⚙️ スタッフ・マスタ管理」モーダル内の「Firestore無料枠 使用量確認」から、
現在の登録件数・画像の推定使用容量・無料枠（合計1GB）に対する残り保存可能枚数の目安を確認できます。
