# セッション記録：Firebaseセットアップ案内

> 日付：2026-05-17  
> 内容：Claude Web からの引き継ぎ受け取り、Firebaseセットアップ手順の案内

---

## 確認した既存コードの概要

- `index.html`：1421行、HTML/CSS/JSが1ファイルに集約
- localStorageキー：`walk-diary-entries`（エントリ一覧）、`walk-diary-settings`（APIキー等）
- 写真はbase64でlocalStorageに直接保存（`entry.image` フィールド）
- エントリのIDは `'e_' + Date.now() + '_' + ランダム5文字`

---

## Firebaseセットアップ手順（ユーザーへ案内済み）

### ステップ1：プロジェクト作成
- [Firebase Console](https://console.firebase.google.com/) でプロジェクト作成
- プロジェクト名例：`walk-diary`

### ステップ2：Authentication
- Google ログインを有効化（サポートメール必要）
- メール/パスワードログインを有効化

### ステップ3：Firestore Database
- ロケーション：`asia-northeast1`（東京）
- テストモードで開始（後でセキュリティルールを厳密化）

### ステップ4：Storage
- ロケーション：`asia-northeast1`
- テストモードで開始

### ステップ5：Webアプリ登録 → firebaseConfig 取得
- コンソール左上⚙「プロジェクトの設定」→「アプリを追加」→ `</>` Web
- 取得した `firebaseConfig` をユーザーからもらって実装に使う

---

## 取得した firebaseConfig

```js
const firebaseConfig = {
  apiKey: "AIzaSyBsQQt6MEZXpAlvm1JWO0092X6OD04xIVI",
  authDomain: "walk-diary-35e01.firebaseapp.com",
  projectId: "walk-diary-35e01",
  storageBucket: "walk-diary-35e01.firebasestorage.app",
  messagingSenderId: "371376455663",
  appId: "1:371376455663:web:08525e754d9cbb938d401a",
  measurementId: "G-ZL99072R5V"
};
```

※ `apiKey` は Firebase の公開識別子であり、HTMLに直書きして問題ない。Gemini APIキーとは別物。

## 次のアクション

- [x] ユーザーから `firebaseConfig` を受け取る
- [x] `index.html` をFirebase対応に改造（フェーズ1実装）
  - Firebase SDK 10.14.1 CDN（modular）を `<script type="module">` で読み込み
  - ローディング画面追加（認証状態確定まで表示）
  - ログイン画面追加（ボタニカルデザインに統一）
    - Googleログイン / メール+パスワード / 新規登録 を切り替え
  - 写真保存を base64/localStorage → Firebase Storage（`photos/{uid}/{docId}.jpg`）に変更
  - エントリ保存を localStorage → Firestore（`entries/{docId}`）に変更
  - 既存localStorageデータのFirestore移行機能（移行済みフラグで再表示防止）
  - 設定画面にサインアウトボタン・アカウント情報追加
  - エクスポートはphotoURLベースのJSONに変更（インポートは廃止）
- [ ] Firestore + Storage セキュリティルール設定（コンソールで手動）
- [ ] GitHubへプッシュして動作確認

## 実装上の注意点

- Firestoreの `where('ownerId') + orderBy('date')` は複合インデックスが必要なため、JSでソートに変更
- `deleteObject` はStorageファイルが存在しない場合エラーになるため try-catch で握りつぶし
- 移行フラグ: `localStorage['walk-diary-migrated-{uid}']` = 'true' でユーザーごとに管理
