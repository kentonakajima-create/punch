# Firebase設定ガイド（ステップ1）

このアプリをFirebaseと連携させるための設定手順です。

## 1. Firebaseプロジェクトの作成

1. [Firebase Console](https://console.firebase.google.com/) にアクセス
2. 「プロジェクトを追加」をクリック
3. プロジェクト名を入力（例：`punch-app`）
4. プロジェクトの作成を完了

## 2. Webアプリの登録

1. Firebase Consoleでプロジェクトを選択
2. 左側の「プロジェクト設定」をクリック
3. 「Webアプリを追加」をクリック
4. アプリニックネーム（例：`web-punch`）を入力して登録
5. 表示されるFirebase設定（SDKセットアップ）をコピー

表示される設定は以下のような形式です：

```javascript
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project-id",
  storageBucket: "your-project.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};
```

## 3. Google認証の有効化

1. Firebase Consoleで「認証」をクリック
2. 「Sign-in method」をクリック
3. 「Google」をクリック
4. 「有効にする」をオンにして保存

## 4. index.htmlに設定を反映

`index.html` の以下の部分を、取得した設定に置き換えます：

```javascript
const firebaseConfig = {
  apiKey: "取得したAPIKeyに置き換え",
  authDomain: "取得したauthDomainに置き換え",
  projectId: "取得したprojectIdに置き換え",
  storageBucket: "取得したstorageBucketに置き換え",
  messagingSenderId: "取得したmessagingSenderIdに置き換え",
  appId: "取得したappIdに置き換え"
};
```

## 5. ローカルテスト（オプション）

ローカルサーバーでテストする場合：

```bash
# Pythonがある場合
python -m http.server 8000

# Node.jsがある場合
npx http-server
```

ブラウザで `http://localhost:8000` を開いてテスト

**注意**: Googleログインはホスト名が `localhost` の場合、`http://localhost:port` でのみ動作します。ホスト名が `127.0.0.1` だと動作しません。

## 6. 本番環境へのデプロイ

Firebase Hostingで無料でホストできます：

```bash
# Firebase CLIをインストール
npm install -g firebase-tools

# プロジェクトにログイン
firebase login

# Firebaseを初期化（プロジェクトを作成したディレクトリで）
firebase init hosting

# デプロイ
firebase deploy
```

## トラブルシューティング

### ログイン画面が表示されたままの場合
- ブラウザの開発者ツール（F12）で、コンソールにエラーが出ていないか確認
- Firebaseの設定が正しく入力されているか確認

### ログインできない場合
- Firebase Console → 認証 → Google が「有効」になっているか確認
- ホスト名が `localhost` であるか確認（`127.0.0.1` は非対応）

### CORS エラーが出る場合
- Firebase Console → 認証 → 設定 → 認可済みドメイン に現在のドメインを追加

---

次のステップ：ステップ2では、出勤・退勤をFirestoreに保存します。
