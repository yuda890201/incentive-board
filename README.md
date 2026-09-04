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
- `tasks/{taskId}` — `{ title, description, reward, repeatMode, repeatIntervalDays, manualText, manualImage, seriesId, spawnedNextTaskId, status, claimedBy, claimedAt, submissionNote, submissionImage, submittedAt, approvedAt, approvedMonth, createdAt }` クエスト掲示板の1件ごとのレコード。`status` は `open`（受注可能）→`claimed`（挑戦中）→`submitted`（承認待ち）→`approved`（達成済み・報酬確定）と遷移します。`repeatMode` は `once`（単発）または `recurring`（繰り返し）
- `competitions/{competitionId}` — `{ title, description, deadline, status, createdAt }` POP/SNSコンペの開催情報。`status` は `open` または `closed`
- `competitionEntries/{entryId}` — `{ competitionId, competitionTitle, staffId, staffName, entryType, image, link, comment, isWinner, prizeAmount, resultMonth, submittedAt }` コンペへの応募1件ごとのレコード。`entryType` が `image` の場合は圧縮画像、`link` の場合はSNS/DriveなどのURLを保存します

`image` はレシート写真・応募作品をリサイズ・JPEG圧縮したData URL文字列です。Firestoreの
1ドキュメントあたり上限（約1MB）に収まるよう、必要に応じて自動的に画質・解像度を落とします。

## 画面構成

ホーム画面に4つの大きな入り口があり、それぞれ独立した画面に遷移します（各画面に専用の🖨印刷ボタンがあります）。
「⚙️ スタッフ・マスタ管理」は右上の小さな歯車アイコンから開くモーダルとして残しています。

- **⚔️ クエスト**: 低頻度タスクの成果報酬型マッチング（後述）
- **🎨 コンペ**: POP/SNS作品コンテスト（後述）
- **🧾 獲得表**: レシート投稿・スタッフごとの獲得件数を月別に集計（従来のメイン画面）
- **🏆 報酬ランキング**: レシート報酬・クエスト報酬・コンペ賞金を合算した**累計支給額のみ**で順位付けする専用ランキング

## クエスト / POP・SNSコンペ

留学生や社保の扶養に入っているスタッフなど、労働時間を抑えたいスタッフ向けに、
時給に依存しない成果報酬型のインセンティブを追加できる機能です。

- **クエスト**: マネージャーが低頻度タスク（清掃など）を報酬額付きで発行 →
  スタッフが早い者勝ちで受注 → 達成報告（メモ＋任意で写真） → マネージャーが承認すると、
  その月の支給額に自動加算されます。
  - **発注タイプ**: 「単発（履歴から再発注可）」または「🔁 繰り返し」を選択できます。
    繰り返しの場合、クリア（承認）から指定日数が経過すると、同じ内容のクエストが自動的に
    再発注されます（自動発注はクエストボードを開いたタイミングでチェックされます。無料の
    Sparkプランのままcronサーバーを使わずに実現するため、サーバー側の定時実行ではなく
    クライアント側でのチェックです）。単発クエストも達成履歴から手動で「🔁 再発注する」
    ことができます。
  - **作業マニュアル**: クエストごとに手順テキスト・参考画像を登録でき、募集中〜達成履歴の
    どの状態でも「📖 マニュアルを見る」から閲覧できます。繰り返し・再発注時もマニュアルは
    引き継がれます。
- **POP/SNSコンペ**: マネージャーがコンペを開催 → スタッフが作品を応募（画像は圧縮してFirestoreに保存、
  動画やSNS投稿はURLで応募） → マネージャー/オーナーが優勝作品を選び賞金額を入力すると、
  その月の支給額に自動加算されます。Storageを使わない構成上、動画ファイル自体のアップロードには対応していません。

「獲得表」「報酬ランキング」の両画面には、レシート実績による金額に加えて、承認済みクエストの報酬額・
コンペの獲得賞金額が自動的に合算されます。「報酬ランキング」は合計支給額のみを順位付け基準にしており、
件数（獲得表）とは独立したランキングです。

## 無料枠の使用量確認

「⚙️」アイコンから開く「スタッフ・マスタ管理」モーダル内の「Firestore無料枠 使用量確認」から、
現在の登録件数・画像の推定使用容量・無料枠（合計1GB）に対する残り保存可能枚数の目安を確認できます。
