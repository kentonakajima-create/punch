# Firestore データベーススキーマ

このドキュメントは、打刻アプリのFirestoreデータベース構造を説明します。

## 全体構成

```
Firestore
├── users (コレクション)
│   ├── {uid1} (ドキュメント)
│   │   ├── name: "田中太郎"
│   │   ├── email: "tanaka@example.com"
│   │   └── punches (サブコレクション)
│   │       ├── doc001: { 出勤記録 }
│   │       ├── doc002: { 退勤記録 }
│   │       └── ...
│   ├── {uid2} (ドキュメント)
│   │   └── punches (サブコレクション)
│   │       └── ...
│   └── ...
```

## ステップ2時点：打刻記録 (punches)

### コレクションパス
```
users/{uid}/punches
```

### ドキュメント構造

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `type` | String | 打刻種別: `"出勤"` \| `"退勤"` \| `"外出"` \| `"戻り"` |
| `time` | String | 打刻時刻（ローカル時刻）: `"HH:MM:SS"` |
| `date` | String | 打刻日付: `"YYYY/MM/DD"` |
| `timestamp` | Timestamp | サーバータイムスタンプ（サーバー側で生成） |
| `createdAt` | Timestamp | ドキュメント作成時刻 |

### 例

```json
{
  "type": "出勤",
  "time": "09:00:00",
  "date": "2026/08/11",
  "timestamp": Timestamp(2026-08-11T00:00:00Z),
  "createdAt": Timestamp(2026-08-11T00:01:23Z)
}
```

## ステップ3予定：ユーザー情報 (users ドキュメント)

将来的には、各ユーザーのドキュメントに基本情報も保存します：

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `name` | String | ユーザー表示名 |
| `email` | String | メールアドレス |
| `role` | String | ユーザーロール: `"user"` \| `"admin"` |
| `createdAt` | Timestamp | ユーザー作成日時 |

## ステップ4予定：申請記録 (applications)

将来的には、申請情報も保存します：

```
users/{uid}/applications
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `type` | String | 申請種別: `"残業申請"` \| `"休日出勤"` \| `"休暇申請"` |
| `status` | String | 申請状態: `"pending"` \| `"approved"` \| `"rejected"` |
| `data` | Object | 申請データ（種別による） |
| `submittedAt` | Timestamp | 申請日時 |

## セキュリティに関する注記

### ステップ2の一時的なセキュリティルール
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;  // テスト用 - 本番禁止
    }
  }
}
```

### ステップ6で実装予定のセキュリティルール
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth.uid == uid;
      
      match /punches/{document=**} {
        allow read, write: if request.auth.uid == uid;
      }
      
      match /applications/{document=**} {
        allow read, write: if request.auth.uid == uid;
      }
    }
  }
}
```

## インデックス

複雑なクエリを実行する場合、Firestoreが自動的にインデックスを提案します。

**ステップ4で必要になるインデックス例：**
- `users/{uid}/punches`: `date` と `timestamp` のインデックス
- クエリ：日付範囲で月間データを取得

## データサイズの目安

1ユーザーが毎日2回（出勤・退勤）打刻した場合：

- 1日：2レコード、約 100 bytes
- 1ヶ月（20営業日）：40レコード、約 2 KB
- 1年：480レコード、約 24 KB
- 5年：2400レコード、約 120 KB

Firestoreの無料枠：50,000 読み取り、20,000 書き込み/月
→ 100ユーザーで毎日1000の打刻でも十分収まります

## バックアップ

Firebase Console → Firestore Database → 「エクスポート」でバックアップ可能。

本番運用時は定期バックアップをお勧めします。

## 次のステップでの変更予定

| ステップ | 変更内容 |
|---------|--------|
| ステップ3 | Firestoreからのデータ読み込み実装 |
| ステップ4 | 申請コレクション追加 |
| ステップ5 | 勤務時間計算フィールド追加 |
| ステップ6 | セキュリティルール本格実装 |
| ステップ7 | 管理者用のロール情報追加 |
