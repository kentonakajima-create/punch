# エラー分析：申請の承認/却下ワークフロー実装

実装日：2026-08-12  
実装内容：申請の承認・却下ボタン追加、確認ダイアログの統一実装

---

## 🔴 発生したエラーと原因分析

### エラー1：承認・却下ボタンが表示されない

**症状：**
- 開発者モードで「承認」「却下」ボタンが画面上に表示されない
- コンソールログには「Button display check」というログが出ていた

**根本原因：**
```
UI状態の不一致：
- 実装者の思考：「監査ログ（page=2）」セクションに表示する
- ユーザーの要望：「申請の履歴」セクション（HOME画面下部）に表示する
→ renderApplications() で正しくボタンを生成していたが、
  ユーザーが見ている「申請の履歴」では非表示だった
```

**解決策：**
- ボタンレンダリングロジックを `renderHistory()` から `renderApplications()` に統一
- UIの責務を明確化：
  - `renderHistory()` = 打刻記録表示
  - `renderApplications()` = 申請履歴表示

**学び（State Sufficiency 原則）：**
> 異なるセクション（監査ログ vs 申請の履歴）に同じ要素を表示する場合は、
> どちらをメインとするか事前に定義すべき

---

### エラー2：取消と却下が同じ扱いになっていた

**症状：**
```javascript
// 問題のあるコード
if (user_cancels) {
  app.canceled = true;  // ユーザーが取消
}
if (admin_rejects) {
  app.canceled = true;  // 管理者が却下
}
// → 両方同じ状態 = 区別不能！
```

**根本原因：**
```
State Sufficiency 違反：
- 概念上は「取消」と「却下」は異なる意味
- しかし実装上は同じフラグ（canceled）で表現
- 結果：「この状態は何か？」が不明確

Timeline:
1. 申請中 (canceled=false, rejected=false)
2. ユーザーが取消 (canceled=true)
3. 「この状態は？」→ canceled=true だけでは「取消」か「却下」か不明
```

**修正前の状態遷移図（欠陥）：**
```
申請中 ──取消--> 取消済み
  │
  └─却下--> ??? (同じ状態)
```

**解決策：**
```javascript
// 状態フラグの分離
canceled: false,  // ユーザーが取消
rejected: false   // 管理者が却下

// 修正後の状態遷移図（正常）
申請中 (canceled=F, rejected=F)
  ├─ユーザー取消 → (canceled=T, rejected=F)
  └─管理者却下 → (canceled=F, rejected=T)
```

**学び（DESIGN_PRINCIPLES.md § State Sufficiency）：**
> 異なる意味状態を同じ内部状態に写してはならない。
> 「取消」と「却下」は意味が異なるので、独立した状態変数が必須。

---

### エラー3：ボタンクリック後、申請オブジェクトが空に見えた

**症状：**
```
コンソール出力：
  Reject clicked for index: 0
  renderApplications called, devMode: true
  app: 
    (オブジェクトが空のように見える)
```

**根本原因：**
```javascript
// 問題のあるコード
for (let i = 0; i < applications.length; i++) {
  const rejectBtn = document.createElement("button");
  rejectBtn.addEventListener("click", function() {
    console.log("app:", applications[i]);  // ← NG！
    // ループ完了後、i = applications.length になっている
    // applications[i] は undefined
  });
}
```

**修正：IIFE（即座実行関数式）パターン**
```javascript
rejectBtn.addEventListener("click", (function(idx) {
  return function() {
    console.log("app:", applications[idx]);  // ← OK！
    // idx は引数で固定される
  };
})(i));  // ← ここで i の値を引数として「閉じ込める」
```

**状態のタイムライン：**
```
ループ実行中（i=0）：
  rejectBtn のイベントハンドラー作成
  → 関数内で i を参照するが、まだ実行されない

ループ完了後（i = applications.length）：
  ユーザーが rejectBtn をクリック
  → イベントハンドラー実行
  → applications[i] を参照
  → i は applications.length のまま = undefined
```

**学び（JavaScript のクロージャー）：**
> ループ内でイベントリスナーを登録する場合、
> ループ変数は関数のスコープ外で捕捉される。
> IIFE で「現在のループ値を引数として固定」することで解決。

---

### エラー4：条件分岐の順序が正しくなかった

**症状：**
```
却下ボタンを3回連打した後：
  却下済み状態なのに「承認/却下」ボタンが表示される
  → 却下を取り消すボタンが表示されない
```

**根本原因：**
```javascript
// 問題のあるコード
if (devMode && !app.canceled) {
  // 承認/却下ボタン表示
}
else if (devMode && app.canceled && !app.rejected) {
  // 復元ボタン表示
}
else if (devMode && app.rejected) {
  // 却下を取り消すボタン表示
}

// 却下済み状態：canceled=false, rejected=true
// → 最初の if (!app.canceled) が true → 承認/却下ボタン表示！（バグ）
```

**修正：条件を特異性の高い順に並べ替え**
```javascript
if (devMode && app.rejected) {
  // 却下を取り消す（最も具体的）
}
else if (devMode && app.canceled && !app.rejected) {
  // 復元（次に具体的）
}
else if (devMode && !app.canceled && !app.rejected) {
  // 承認/却下（最も一般的）
}
```

**学び（条件分岐の設計）：**
> 複数の独立状態フラグがある場合、
> より「具体的」な状態から「一般的」な状態へ順序付けすること。
> さもなければ、後ろの条件が死コード化する。

---

### エラー5：申請承認時の確認ダイアログが出ないことがあった

**症状：**
```
申請の承認ボタンをクリック
→ ダイアログが表示されずに直接承認される
（ただし、コンソールは何も出力しない）
```

**根本原因：**
```
当初は、以下の2つのシナリオが混在していた：

【申請履歴（applications）の承認】
  → openApproveConfirm() で正しくダイアログ表示
  
【本日の打刻（history）の承認】
  → 直接実行 item.pending = false
  → ダイアログなし

ユーザーが「承認ボタンだけポップアップが出てこない」と報告
→ 「本日の打刻」の承認機能に対する要望だった
```

**修正：両方にダイアログを実装**
```javascript
// 申請の承認
approveBtn.addEventListener("click", (function(idx) {
  return function() {
    openApproveConfirm(idx, applications[idx]);  // ← ダイアログ表示
  };
})(i));

// 打刻の承認（修正：以前は直接実行）
confirmBtn.addEventListener("click", function () {
  openPunchApproveConfirm(index, item);  // ← ダイアログ表示に変更
});
```

**学び（複数の「承認」概念の違い）：**
> 「申請の承認」と「打刻の承認」は別の概念。
> ユーザーが「承認ボタン」と言ったときに、どちらを指しているか確認が重要。

---

### エラー6：本日の打刻の取消・復元が一撃で実行される

**症状：**
```
打刻記録の「取消」ボタンをクリック
→ 確認ダイアログが表示されずに、すぐに状態が変わる
→ ユーザーの意図に反する可能性
```

**根本原因：**
```javascript
// 問題のあるコード
removeBtn.addEventListener("click", function () {
  item.canceled = true;          // ← 確認なし！
  addLog(item, "取消");
  renderHistory();
});
```

**修正：確認ダイアログを追加**
```javascript
removeBtn.addEventListener("click", function () {
  openCancelConfirm(index, item);  // ← ダイアログ表示
});

function openCancelConfirm(index, item) {
  // ダイアログ表示
  // ユーザーが「取消する」をクリック時にのみ実行
}
```

**学び（UI の一貫性）：**
> 重要な操作（状態変更）は、打刻の「確認」ダイアログと同じレベルの確認が必要。
> 「取消」も「削除」と同様に取り返しがつきにくい操作。

---

### エラー7：承認後に復元ボタンを押すと承認がリセットされた

**症状：**
```
Timeline:
1. 打刻記録が pending=true （未承認）
2. 「承認」ボタンをクリック → pending=false に変更
3. 「復元」ボタンをクリック → canceled=false に変更
4. その後、画面を見ると pending=true に戻っている（未承認状態）
```

**根本原因：**
```javascript
// 問題のあるコード
document.getElementById("punchApproveOkBtn").addEventListener("click", function() {
  item.pending = false;           // ← ローカルのみ変更
  addLog(item, "申請を確定");
  renderHistory();
  // ← Firestore には保存されない！
  
  // その後、Firestore のリスナーが古いデータで上書き
  // pending=true のデータが再度ローカルに反映される
});
```

**問題の流れ：**
```
Local:     pending=true
Firebase:  pending=true

↓ 承認ボタンをクリック

Local:     pending=false     ← 変更
Firebase:  pending=true      ← 変更されない

↓ Firestore リスナーが発火

Local:     pending=true      ← Firebase から古いデータで上書き
Firebase:  pending=true
```

**修正：Firestore に明示的に保存**
```javascript
document.getElementById("punchApproveOkBtn").addEventListener("click", function() {
  item.pending = false;
  addLog(item, "申請を確定");
  
  // ← ここを追加
  if (item.docId && isFirebaseConfigured && currentUser && db) {
    db.collection("users").doc(currentUser.uid)
      .collection("punches").doc(item.docId)
      .update({ pending: false })  // ← Firebase に保存
      .catch((error) => console.error("Firestore更新エラー:", error));
  }
  
  renderHistory();
});
```

**学び（Firestore との同期問題）：**
> ローカルとFirestoreを使い分ける際、どちらが「単一の真実」かを明確にすべき。
> 本システムでは Firestore が真実のため、ローカルの変更は必ず Firebase に保存すること。

---

## 📊 エラー分類と優先度

| # | エラー | 種類 | 根本原因 | 優先度 |
|---|--------|------|--------|--------|
| 1 | ボタン非表示 | UI | 表示セクション不一致 | 🔴 機能 |
| 2 | 取消≒却下 | 設計 | State Sufficiency 違反 | 🔴 設計 |
| 3 | オブジェクト空 | JS | クロージャーキャプチャ | 🟡 デバッグ |
| 4 | 条件分岐バグ | ロジック | 条件順序 | 🔴 ロジック |
| 5 | ダイアログ不表示 | 実装 | 複数概念の混同 | 🟡 UX |
| 6 | 一撃実行 | UX | 確認なし | 🟡 安全性 |
| 7 | 承認リセット | データ | Firebase未保存 | 🔴 データ |

---

## 🎯 得られた教訓

### 1. **State Sufficiency（状態十分性）**
- ❌ 「取消」と「却下」を同じフラグで表現しない
- ✅ 異なる意味 = 異なる状態変数

### 2. **JavaScript のクロージャー**
- ❌ ループ内でループ変数を遅延キャプチャしない
- ✅ IIFE で現在の値を「固定」する

### 3. **条件分岐の設計**
- ❌ 一般的な条件を最初に置かない
- ✅ より具体的な条件から順に並べる

### 4. **複数の「承認」概念**
- ❌ 「申請の承認」と「打刻の承認」を混同しない
- ✅ ドメイン用語を区別して実装する

### 5. **ローカル vs Firestore**
- ❌ ローカルだけ変更して Firestore を無視しない
- ✅ 変更時は常に Firestore に保存する

### 6. **UI の一貫性**
- ❌ 重要な操作によって、ある時は確認ダイアログ、ある時はなしにしない
- ✅ 取り返しのつかない操作は常に確認ダイアログを表示

---

## 📈 実装過程の改善点

### 改善が機能した例

| 改善 | 効果 | 学習 |
|------|------|------|
| `rejected` フラグの追加 | 状態を明確に区別可能に | 状態設計の重要性 |
| IIFE パターン | イベントハンドラーが正しく動作 | JS のスコープ/クロージャー |
| Firebase 保存の追加 | ローカル-Firebase 同期が正常化 | 単一の真実の原則 |
| 確認ダイアログの統一 | UX が一貫性を持つ | 概念の統一 |

### 今後の予防策

1. **実装前の状態設計**
   - 状態遷移図を描く
   - 各状態で可能な操作を定義
   - 不正な状態が表現不能か確認

2. **ループ内のイベントリスナー**
   - テンプレート：IIFE で変数をキャプチャ
   - コード レビュー時に必ず確認

3. **Firestore 同期**
   - 「ローカルのみ変更」は禁止
   - リスナーで上書きされるリスク = バグ

4. **複数ドメイン概念**
   - 用語集を作成（「承認」の定義）
   - 変数名は概念をはっきり表現

---

## 🔗 関連ドキュメント

- `DESIGN_PRINCIPLES.md` - 設計原則（State Sufficiency など）
- `CODE_REVIEW.md` - コード品質分析
- `FIREBASE_SETUP.md` - Firestore 設計

---

**作成日：** 2026-08-12  
**対象コミット：** b0e3568  
**実装者：** Claude Code
