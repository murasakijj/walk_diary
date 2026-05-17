# 散歩図鑑（Walk Diary）プロジェクト引き継ぎ書

> Claude Web → Claude Code への引き継ぎ（2026-05-17）

---

## プロジェクト概要

散歩中に出会った花・植物・犬などをスマホで撮影し、AI（Google Gemini API）で名前を自動識別して記録するWebアプリ。ボタニカル図鑑風のデザイン。

### 現在の状態

- **動作中**：単一のHTMLファイル（`index.html`）として完成
- **配信先**：GitHub Pages（Publicリポジトリ）
- **データ保存**：ブラウザのlocalStorage（端末内のみ）
- **AI識別**：Google Gemini API（`gemini-2.5-flash`モデル）
- **UI**：和風ボタニカル図鑑デザイン（Shippori Mincho、Cormorant Garamond等のフォント）

### 公開URL

**GitHub Pages**: https://murasakijj.github.io/walk_diary/

### ファイル構成

```
C:\workspace\walk_diary\
├─ index.html        ← 単一ファイルにHTML/CSS/JS全部入り
├─ docs/
│   └─ handoff.md   ← このファイル
└─ README.md
```

### デザイントークン（CSS変数）

```css
--ink: #2a2520
--paper: #f5efe4
--paper-deep: #ebe2d2
--paper-shadow: #ddd2bc
--moss: #4a5d3a
--moss-light: #6b7d52
--leaf: #5a7a3a
--bark: #6b4f3a
--berry: #a04a3a
--gold: #b89456
--sky: #8ba89c
```

---

## 既存の機能

1. **記録機能**
   - カメラ起動 or 写真選択
   - Gemini APIで自動識別（名前・学名・分類・解説）
   - GPS自動取得 + OpenStreetMap Nominatimで地名変換
   - 手動編集可能

2. **3つのビュー**
   - ギャラリー（Masonryレイアウト、タイル状）
   - 地図（Leaflet + OpenStreetMap、ピン表示）
   - タイムライン（日付グループ）

3. **詳細・編集・削除**
   - カードタップで詳細モーダル
   - 編集ボタンで内容更新
   - 削除ボタン

4. **設定**
   - Gemini APIキー設定（localStorage保存）
   - 自動識別ON/OFF
   - データのエクスポート/インポート（JSON）

5. **エラーハンドリング**
   - APIエラーの詳細を画面に表示する仕組みあり（デバッグしやすい）

---

## フェーズ1：Firebase移行（現在着手中）

### ゴール

ローカルストレージから、Firebaseクラウド保存に切り替える。
家族・友人3〜5人で使えるよう、ユーザー認証も導入する。

### 共有スタイル

- 各自が**個人アルバム**を持つ
- 見せたい時に**共有リンク**で他人に見せられる（フェーズ2で実装）
- 「誰が投稿したか」が見える設計

### システム構成

```
ユーザー（スマホブラウザ）
  ↓
┌──────────────┬─────────────────────┐
│ GitHub Pages │ Firebase            │
│ (HTML配信)   │  - Authentication   │
│              │  - Storage (写真)    │
│              │  - Firestore (DB)   │
└──────────────┴─────────────────────┘
                  ↓
              Gemini API（AI識別）
```

### 認証方式

- Googleログイン
- メールアドレス＋パスワードログイン
- 両方選べるようにする

### Firestoreデータ構造

```
users/{userId}
  ├ name: string
  ├ photoURL: string
  └ createdAt: timestamp

entries/{entryId}
  ├ ownerId: string         ← 投稿者
  ├ ownerName: string
  ├ name: string             ← 例：ヒメジョオン
  ├ scientificName: string
  ├ category: string         ← 花/植物/樹木/犬/鳥/昆虫/その他
  ├ description: string
  ├ location: string
  ├ coords: { lat, lng } | null
  ├ photoURL: string         ← Firebase Storage URL
  ├ date: timestamp
  ├ visibility: string       ← "private" | "shared"
  └ shareToken: string | null

shares/{shareToken}（フェーズ2で実装）
  ├ entryIds: string[]
  ├ ownerId: string
  ├ createdAt: timestamp
  └ expiresAt: timestamp | null
```

### Firebase Storageパス構造

```
photos/{userId}/{entryId}.jpg
```

### Firebaseセキュリティルール

```js
// Firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }
    match /entries/{entryId} {
      allow read: if request.auth.uid == resource.data.ownerId
                   || resource.data.visibility == 'shared';
      allow create: if request.auth.uid == request.resource.data.ownerId;
      allow update, delete: if request.auth.uid == resource.data.ownerId;
    }
  }
}

// Storage
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /photos/{userId}/{fileName} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }
  }
}
```

---

## 実装プラン

### フェーズ1（現在）：認証＋クラウド保存

- [ ] Firebaseプロジェクト作成（ユーザーが手動実施）
- [ ] Authentication有効化（Google + メール/パスワード）
- [ ] Firestore有効化
- [ ] Storage有効化
- [ ] Webアプリ登録 → `firebaseConfig` 取得・共有
- [ ] Firebase SDK組み込み（CDN経由）
- [ ] ログイン画面の追加
- [ ] 写真アップロード処理をFirebase Storageに切り替え
- [ ] 記録の保存/読み込みをFirestoreに切り替え
- [ ] 既存localStorageデータのFirestore移行機能
- [ ] セキュリティルール設定（Firestore + Storage）
- [ ] GitHub Pushして動作確認

### フェーズ2（後で）：共有機能

- 共有リンク作成UI
- 共有ビュー画面（ログイン不要で閲覧可能）
- 公開/非公開の切り替え

### フェーズ3（任意）：体験向上

- 投稿者の名前・アイコン表示
- 共有リンクの有効期限
- フィルタ・検索機能

---

## 開発上の重要事項

### コードスタイル

- 単一HTMLファイル維持（ビルド不要）
- Firebase SDKはCDN経由で読み込み（modular SDK v10+推奨）
- 既存のデザイン（ボタニカル和風）は崩さない
- 既存のCSS変数を活用

### セキュリティ

- Firebase Configの`apiKey`はHTMLに直書きOK（公開設定値、Gemini APIキーとは別物）
- Gemini APIキーはユーザーが自分で設定（localStorage、外部送信なし）
- Firestoreセキュリティルールで「自分のデータのみ読み書き可能」を厳密に設定する
- Storageセキュリティルールも同様

### 既存データの扱い

- localStorageに既にデータがある人向けに、**初回ログイン時にFirestoreへ移行する機能**を入れる
- 移行後、ローカルデータは保持（バックアップとして）

---

## 過去の主なエラーと解決履歴

| エラー | 原因 | 解決方法 |
|--------|------|---------|
| アーティファクト内でClaude API呼び出し不可 | 制限 | Gemini APIに変更 |
| `edge://external-file/` でCORS制限 | ローカルファイル | GitHub Pages公開で解決 |
| `gemini-2.0-flash` がHTTP 429 | 廃止モデル | `gemini-2.5-flash`に変更 |

---

## ユーザー情報

- スマホ：iPhone
- 開発環境：Windows PC、VSCode、`C:\workspace\walk_diary\`
- GitHub Pagesで既にアプリを公開中
