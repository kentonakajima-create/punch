# ステップ2実装内容：出勤・退勤をFirestoreに保存

## ✅ 実装完了した内容

### 1. Firestore参照の作成
```javascript
let db = null;
if (isFirebaseConfigured) {
  firebase.initializeApp(firebaseConfig);
  auth = firebase.auth();
  db = firebase.firestore();  // ここで初期化
}
```

### 2. 打刻データをFirestoreに保存する関数
```javascript
function savePunchToFirestore(type, time, day) {
  if (!isFirebaseConfigured || !currentUser || !db) return Promise.resolve();

  const timestamp = getNow();
  return db.collection("users").doc(currentUser.uid)
    .collection("punches").add({
      type: type,          // "出勤" | "退勤" | "外出" | "戻り"
      time: time,          // "HH:MM:SS"
      date: day,           // "YYYY/MM/DD"
      timestamp: firebase.firestore.Timestamp.fromDate(timestamp),
      createdAt: firebase.firestore.Timestamp.fromDate(timestamp)
    });
}
```

### 3. 打刻確定時にFirestoreに保存
以下のすべての打刻確定処理でFirestoreに保存されます：

- **通常の打刻確定** → `confirmPunch()`
- **不足分を補完してから打刻** → `fillDone()`
- **警告を無視して打刻** → `fixForce()`

### 4. Firestoreデータ構造

打刻データはユーザーIDに紐づけて保存されます：

```
Firestore
└── users (コレクション)
    └── {uid} (ドキュメント - ユーザーのUID)
        └── punches (サブコレクション - 打刻記録)
            └── {documentId} (自動生成)
                ├── type: "出勤"
                ├── time: "09:00:00"
                ├── date: "2026/08/11"
                ├── timestamp: (Firestoreタイムスタンプ)
                └── createdAt: (Firestoreタイムスタンプ)
```

## 🔄 動作フロー

1. ユーザーが「出勤」などのボタンを押す
2. 確認ダイアログが表示される
3. ユーザーが「確定する」をクリック
4. ローカルの `history` 配列に追加（即座に画面反映）
5. 同時に Firestore に非同期で保存（バックグラウンド）
6. 保存失敗時にアラート表示

## 🔑 重要なポイント

### ユーザーID (uid) による紐づけ
- 各ユーザーのデータは `currentUser.uid` で分離
- 他のユーザーのデータは絶対に混ざらない
- 次ステップのセキュリティルール設定で強制化

### オンライン/オフライン対応
- ローカル配列に先に追加 → 即座にUI更新
- Firestoreに非同期保存 → バックグラウンドで実行
- 保存失敗時にアラート → ユーザーに通知
- ネット復帰時に自動同期できる基盤

### タイムスタンプ
- `timestamp`: 打刻のサーバータイムスタンプ（サーバー側で生成）
- `createdAt`: ドキュメント作成時刻

## 🧪 テスト方法

### 1. Firebase設定確認
`FIREBASE_SETUP.md` に従ってFirebaseプロジェクト設定済みか確認

### 2. Firestoreデータベースの有効化
1. Firebase Console → 「Firestore Database」
2. 「データベースを作成」をクリック
3. 場所：Tokyo（日本）を選択
4. セキュリティルール：テスト用に設定（本番前に変更必須）

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

### 3. 打刻してみる
1. ブラウザで開く
2. Googleでログイン
3. 「出勤」ボタンをクリック
4. 「確定する」をクリック
5. Firebase Console → Firestore → `users/{yourUID}/punches` でデータが表示されるか確認

## 📊 Firebase Consoleで確認

### Firestoreデータの確認方法
1. [Firebase Console](https://console.firebase.google.com/) でプロジェクト選択
2. 左側の「Firestore Database」をクリック
3. `users` → `{あなたのUID}` → `punches` をクリック
4. 追加された打刻レコードが表示される

### 保存エラーの確認
1. ブラウザの開発者ツール（F12）を開く
2. コンソールでエラーが出ていないか確認
3. Firestoreへのアクセス権限エラーが出ていないか確認

## 🚨 よくある問題

### 「Permission denied」エラー
**原因**: Firestoreのセキュリティルールが制限的
**解決**: Firebase Console → Firestore → ルール → テスト用ルールに変更

### データが保存されない
1. ブラウザコンソールでエラーを確認
2. ネットワークタブで Firestore リクエストを確認
3. Firebase Console でプロジェクトが正しく設定されているか確認

### ローカルにだけデータが表示される
1. Firestore有効化されているか確認
2. セキュリティルール設定されているか確認
3. ブラウザコンソールでエラーを確認

## 📋 実装コード例

### 打刻確定の完全フロー
```javascript
function confirmPunch() {
  if (pending === null) return;
  
  // 1. ローカル履歴に追加
  const item = makePunch(pending.type, pending.time, pending.day, false, false, "作成（" + pending.time + "）");
  history.push(item);
  
  // 2. Firestoreに保存（非同期）
  savePunchToFirestore(pending.type, pending.time, pending.day);
  
  // 3. UI更新して確認ダイアログを閉じる
  renderHistory();
  closeConfirm();
}
```

## ⏭️ 次のステップ

**ステップ3**: 今日の打刻状態をFirestoreから読み込んで表示

```
今日すでに出勤打刻したかどうかが分かるように、
現在の状態を画面に表示して。
出勤済みなら「出勤中」、退勤済みなら「退勤済み」のように。
```

このステップで、Firestoreから履歴を読み込む処理を実装します。
