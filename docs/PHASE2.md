# 散歩図鑑（Walk Diary）フェーズ2 仕様書

## このドキュメントの位置付け

フェーズ1（Firebase認証＋クラウド保存）完了後、次に取り組む共有機能の仕様書です。

**前提**：フェーズ1が完了している（Firebase Auth/Firestore/Storage導入済み、Googleログイン＋メール/パスワードログイン動作中、写真と記録のクラウド保存動作中）

---

## フェーズ2のゴール

**自分の記録を、見せたい相手にだけ共有リンクで見せられるようにする。**

### ユースケース

- 「今日見つけたヒメジョオン、お母さんに見せたい」→ 1件だけ共有
- 「今月の散歩記録を友達にまとめて見せたい」→ 複数件まとめて共有
- 「家族には散歩記録を全部見てほしい」→ 全件共有
- 「やっぱり見せたくなくなった」→ 共有を無効化

### 共有を受け取った側の体験

- LINEなどで送られてきたURLをタップ
- ログイン不要で開ける
- 共有された記録だけが、専用のビューで表示される
- 投稿者の名前が表示される（「○○さんが見つけたもの」）
- 読み取り専用（編集・削除はできない）

---

## 機能要件

### 1. 共有リンク作成UI

#### 1-A. 単一記録の共有
- 詳細モーダル（記録タップで開くやつ）に「共有する」ボタンを追加
- ボタン位置：既存の「編集する」「削除する」と並べる
- タップで共有設定モーダルを開く

#### 1-B. 複数記録の共有
- ギャラリービュー上部に「選択モード」トグルを追加
- 選択モード中はカードをタップで選択（チェックマーク表示）
- 下部に「選択中のN件を共有」ボタンが出現
- タップで共有設定モーダルを開く

#### 1-C. 全件共有
- 設定モーダル内に「すべての記録を共有」オプションを追加
- 注意喚起：「今後追加する記録もすべて見られます」

### 2. 共有設定モーダル

共有リンクを作成する際に表示するモーダル。以下を設定可能：

- **有効期限**
  - 1日 / 1週間 / 1ヶ月 / 無期限から選択
  - デフォルト：1週間
- **共有対象**（自動表示、編集不可）
  - 「1件の記録」「N件の記録」「全件＋今後の記録」のいずれか
- **作成ボタン** → タップでトークン生成＆共有URL表示

### 3. 共有URL表示

共有作成後の画面：
- URLをテキストフィールドで表示
- 「コピー」ボタン（クリップボードへコピー）
- 「LINEで送る」「メールで送る」などの共有ボタン（Web Share API使用）
- QRコードも表示するとモバイル間で便利（オプション）

### 4. 共有管理画面

設定モーダル内に「共有中のリンク一覧」セクションを追加：

- 作成日時、有効期限、対象件数、状態（有効/期限切れ）を一覧表示
- 各リンクに「URLをコピー」「無効化（削除）」ボタン
- 一目で「今どれだけ共有してるか」がわかる

### 5. 共有ビュー画面

URLに `?share=xxx` がついている時の特別な表示モード：

#### 5-A. レイアウト
- ヘッダーは既存と同じ（「散歩図鑑」ロゴ）
- サブヘッダーに「○○さんが見つけたもの」と投稿者名表示
- 「全N件の出会い」と件数表示
- 既存のギャラリービューをそのまま使う（タイル状）
- 記録タップで詳細表示（読み取り専用）

#### 5-B. 機能制限
- ログイン画面はスキップ
- 「記録する」ボタンは非表示
- 詳細モーダルの「編集」「削除」ボタンは非表示
- 地図ビュー・タイムラインビューも表示OK（位置情報がある場合）
- 設定ボタンは非表示

#### 5-C. 「自分も使ってみたい」誘導
- 画面下部に「あなたも散歩図鑑を始めてみませんか？」リンク
- タップでログイン画面へ（共有閲覧者を新規ユーザーに）

### 6. 期限切れ・無効化時の表示

- 期限切れまたは無効化されたリンクを開いた場合：
  - エラー画面：「このリンクは期限切れか、無効化されました」
  - 散歩図鑑のロゴと簡単な説明
  - 「あなたも始めてみませんか」リンク

---

## データ構造

### Firestore: `shares` コレクション

```
shares/{shareToken}
  ├ ownerId: string              # 共有者のUID
  ├ ownerName: string            # 表示用の名前（キャッシュ）
  ├ shareType: string            # "single" | "multiple" | "all"
  ├ entryIds: string[] | null    # 指定共有の場合のentry IDリスト
                                  # shareType="all" の場合は null
  ├ createdAt: timestamp
  ├ expiresAt: timestamp | null  # null = 無期限
  └ revoked: boolean             # 無効化フラグ（デフォルト false）
```

**shareTokenの生成方法**：
- 推測されにくいランダム文字列（24文字以上推奨）
- `crypto.randomUUID().replace(/-/g, '')` で32文字の16進数文字列

### entries の更新

既存の `entries/{entryId}` に以下を追加：

```
entries/{entryId}
  ├ ...(既存フィールド)
  ├ visibility: string           # "private"（デフォルト） | "shared"
  └ sharedIn: string[]           # このentryが含まれるshareTokenのリスト
```

---

## Firestoreセキュリティルール（更新版）

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }

    match /entries/{entryId} {
      allow read: if (request.auth != null && request.auth.uid == resource.data.ownerId)
                   || resource.data.visibility == 'shared';
      allow create: if request.auth != null && request.auth.uid == request.resource.data.ownerId;
      allow update, delete: if request.auth != null && request.auth.uid == resource.data.ownerId;
    }

    match /shares/{shareToken} {
      allow read: if true;
      allow create: if request.auth != null && request.auth.uid == request.resource.data.ownerId;
      allow update, delete: if request.auth != null && request.auth.uid == resource.data.ownerId;
    }
  }
}
```

---

## 実装の優先順位と進捗

### Must（実装済み 2026-05-17）
- [x] 単一記録の共有リンク作成
- [x] 共有ビュー画面の表示（読み取り専用）
- [x] shares コレクションの作成・読み取り
- [x] セキュリティルール更新

### Should（実装済み 2026-05-17）
- [x] 有効期限の設定（1日/1週間/1ヶ月/無期限）
- [x] 共有リンクの無効化
- [x] 共有管理画面（設定内一覧）
- [x] Web Share API（ネイティブ共有）

### Nice to have（未実装）
- [ ] 複数記録の選択モード
- [ ] 全件共有
- [ ] QRコード表示
- [ ] 期限切れ画面のおしゃれ化

---

## ユーザー情報

- 共有予定人数：3〜5人（家族や親しい友人）
- 共有相手のITリテラシー：一般的なスマホユーザー（LINEは使える）
- セキュリティ要件：「URLを知らない人にはアクセスされない」程度でOK
