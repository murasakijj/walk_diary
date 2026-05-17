# Firebaseセキュリティルール設定手順

> アプリURL: https://murasakijj.github.io/walk_diary/  
> Firebaseプロジェクト: walk-diary-35e01

**注意：** Firebase Storageは使用しない構成に変更済み（写真はbase64でFirestoreに直接保存）。  
設定が必要なのは **FirestoreとAuth の2箇所** のみ。

---

## 1. Firestoreのセキュリティルール

### 手順

1. **Firebase Consoleを開く**  
   https://console.firebase.google.com/ → プロジェクト「walk-diary-35e01」を選択

2. **左メニューから「Firestore Database」をクリック**

3. **上部のタブから「ルール」をクリック**  
   （「データ」タブの隣にあります）

4. **下記のルールを丸ごとコピーして貼り付ける**（既存の内容は全部消してOK）

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ユーザー情報：ログイン済みなら読める、自分のものだけ書ける
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }

    // 記録エントリ：自分のものだけ読み書き・削除できる
    match /entries/{entryId} {
      allow read:   if request.auth.uid == resource.data.ownerId;
      allow create: if request.auth.uid == request.resource.data.ownerId;
      allow update, delete: if request.auth.uid == resource.data.ownerId;
    }

  }
}
```

5. **「公開」ボタンをクリック** → 「公開しますか？」が出たら「公開」

---

## 2. Authenticationの承認済みドメイン設定

GitHub PagesのURLをFirebase Authの許可リストに追加しないと、  
本番環境でGoogleログインが「このドメインは許可されていません」エラーになります。

1. **左メニューから「Authentication」をクリック**

2. **「Settings」タブをクリック**（「Sign-in method」タブの隣）

3. **「承認済みドメイン」セクションを見つける**

4. **「ドメインを追加」をクリック**して以下を入力：
   ```
   murasakijj.github.io
   ```

5. **「追加」をクリック**

---

## 設定後の確認

上記3つが完了したら：

1. https://murasakijj.github.io/walk_diary/ を開く
2. Googleログインを試す
3. 写真を1枚記録して保存されることを確認
4. Firebase Console → Firestore → 「データ」タブに記録が増えていることを確認

---

## トラブル時のよくあるエラー

| エラー内容 | 原因 | 対処 |
|-----------|------|------|
| 「このドメインは許可されていません」 | Authドメイン未設定 | 手順3を実施 |
| 「権限がありません」 | セキュリティルール未設定 or テストモード期限切れ | 手順1・2を実施 |
| 写真が表示されない | StorageルールでReadが拒否 | 手順2を確認 |
| ログイン後に真っ白 | JSエラーの可能性 | ブラウザの開発者ツール（F12）→ Consoleを確認 |
