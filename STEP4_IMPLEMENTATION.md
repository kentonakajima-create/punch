# ステップ4実装内容：過去の打刻履歴をFirestoreから読み込んで表示

## ✅ 実装完了した内容

### 1. Firestoreクエリの変更

**変更前（ステップ3）：本日のデータのみ**
```javascript
db.collection("users").doc(uid)
  .collection("punches")
  .where("date", "==", today)        // 本日だけ
  .orderBy("timestamp", "asc")       // 古い順
  .onSnapshot(...)
```

**変更後（ステップ4）：全期間のデータ**
```javascript
db.collection("users").doc(uid)
  .collection("punches")
  // where句を削除 → 全期間のデータ取得
  .orderBy("timestamp", "desc")      // 新しい順
  .onSnapshot(...)
```

### 2. クエリの意味

| 項目 | 値 | 説明 |
|------|-----|------|
| コレクション | `punches` | 全打刻データ |
| where条件 | なし | 全期間 |
| 並び順 | `timestamp desc` | 新しい順（最新が最初） |
| リアルタイム | ✅ | Firestoreの変更を自動反映 |

### 3. 日付ごとのグループ化

既存の `renderHistory()` 関数が、`page = 1` のときに自動的に日付でグループ化します：

```javascript
function renderHistory() {
  if (page === 0) {
    // 本日のデータのみ表示
  } else if (page === 1) {
    // 日付でグループ化して全履歴を表示
    let lastDay = null;
    for (let i = 0; i < history.length; i++) {
      const item = history[i];
      if (item.day !== lastDay) {
        // 日付ヘッダーを追加
        const heading = document.createElement("div");
        heading.className = "dayHeading";
        heading.textContent = item.day;
        records.appendChild(heading);
        lastDay = item.day;
      }
      records.appendChild(makeRow(item, i));
    }
  }
}
```

### 4. ページング機能

既存のボタン機能で、本日と全履歴を切り替え可能：

```
本日の打刻 ▶ (クリック)
     ↓
全履歴 ◀ (クリック)
     ↓
本日の打刻 ▶
```

---

## 🔄 動作フロー

### ログイン時：
```
Googleでログイン
  ↓
currentUser が設定される
  ↓
loadAllPunchesFromFirestore() が実行
  ↓
Firestoreから全期間のデータを取得（新しい順）
  ↓
history に反映
  ↓
renderHistory() で画面更新（本日表示）
```

### 「全履歴 ▶」ボタンをクリック：
```
page = 0 → page = 1
  ↓
renderHistory() が日付ごとにグループ化
  ↓
全期間のデータが日付別に表示される
  ↓
新しい日付が上に来る（最新から過去へ）
```

### Firestoreにデータが追加されたとき：
```
Firestoreの変更を検知
  ↓
onSnapshot() のコールバックが自動実行
  ↓
history が更新される
  ↓
renderHistory() で画面自動更新
```

---

## 📊 データ構造

Firestore に保存されるデータ（各打刻レコード）：

```javascript
{
  type: "出勤",           // 出勤・退勤・外出・戻り
  time: "09:00:00",       // HH:MM:SS
  date: "2026/08/11",     // YYYY/MM/DD
  timestamp: Timestamp,   // サーバータイムスタンプ（ソート用）
  createdAt: Timestamp    // 作成時刻
}
```

---

## 🧪 動作確認

### 1. 複数日の打刻を作成

```
今日（2026/08/11）：「出勤」「退勤」
昨日（2026/08/10）：「出勤」「退勤」
一昨日（2026/08/09）：「出勤」「退勤」
```

### 2. ホーム画面で確認

- ✅ 本日のデータが表示される
- ✅ ステータスバーが「勤務中」「退勤済み」などに更新される

### 3. 「全履歴 ▶」をクリック

- ✅ 全期間のデータが日付ごとに表示される
- ✅ 最新の日付が上に来ている
- ✅ 各日付内で、古い順に表示される

### 4. Firebase Consoleで確認

- Firestore → `users/{uid}/punches`
- 複数のレコードが保存されているか確認
- `timestamp` が新しい順に並んでいるか確認

### 5. 「◀ 本日」をクリック

- ✅ 本日のデータのみに戻る

---

## 🔑 重要なポイント

### リアルタイム同期

`onSnapshot()` を使用しているため：
- 別のタブから打刻するとリアルタイムで反映される
- Firestoreに追加されたデータは即座に表示される

### メモリ効率

全期間のデータをローカルメモリに保持しているため：
- データが多い場合（数年分）はスクロールが遅くなる可能性
- 次のステップ（ステップ5以降）でページング実装を検討

### 日付でのグループ化

```javascript
if (item.day !== lastDay) {
  // 日付が変わったら、ヘッダーを追加
}
```

この仕組みにより、同じ日付のデータは自動的にグループ化される。

---

## 📈 次のステップ

**ステップ5：勤務時間を計算して表示**

```
各日の出勤時刻と退勤時刻から勤務時間を計算して、
履歴の各行に表示する
```

このステップで、表示されたデータから「勤務時間」を算出します。

例：
```
出勤 09:00
退勤 18:30
     ↓
勤務時間：9時間30分
```

---

## ⚠️ 注意事項

### オンライン/オフライン対応

現在の実装は、ネットワークが繋がっていることを前提としています。

オフライン時の対応は、次のステップで検討します。

### セキュリティ

本番環境では、ステップ6で以下のセキュリティルールを設定します：

```javascript
match /users/{uid}/punches/{document=**} {
  allow read, write: if request.auth.uid == uid;
}
```

これにより、ユーザーは自分のデータのみにアクセス可能になります。
