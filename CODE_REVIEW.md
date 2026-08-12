# コード欠陥分析：3つの設計原則に基づく

日付：2026-08-12  
対象ファイル：`index.html`  
分析方法：DESIGN_PRINCIPLES.md の3つの公理に照らし合わせ

---

## 🔴 重大な欠陥（即座に改善推奨）

### 1. 【State Sufficiency 違反】UI状態が散在している

**問題の本質：**
```javascript
// 現在の状態変数（断片化）
let pending = null;          // 打刻確認中？
let fillType = null;         // 補完中？
let editIndex = null;        // 編集中？
let deleteIndex = null;      // 削除確認中？
let fixMode = false;         // 修正モード？
let page = 0;                // 表示ページ（0=本日, 1=全履歴, 2=完全履歴）
let currentApplyType = null; // 申請中？

// UIの表示状態（複数の要素が独立）
#homeView.hide
#applyView.show
#overlay.show
#dialog（確認/編集/削除エリア）
#settingsPanel.show
#loginView.show
```

**欠陥：** 上記の状態変数のいずれかを見ただけでは、アプリ全体の状態が分からない。

例：
```javascript
pending !== null && editIndex !== null  // 矛盾状態？
fixMode === true && overlay.show() が false  // 矛盾？
page === 0 && applyView.show  // 矛盾？
```

**DESIGN_PRINCIPLES.md との照合：**
> 異なる意味状態を同じ内部状態に写してしまっている

**根治策：**
```javascript
// 統一された状態管理
const UIState = {
  HOME: "home",                    // ホーム画面
  PUNCH_CONFIRM: "punch_confirm",  // 打刻確認ダイアログ
  PUNCH_EDIT: "punch_edit",        // 打刻編集ダイアログ
  PUNCH_DELETE: "punch_delete",    // 削除確認ダイアログ
  APPLY_FORM: "apply_form",        // 申請フォーム
  SETTINGS: "settings",            // 設定パネル
  LOGIN: "login"                   // ログイン画面
};

let currentUIState = UIState.HOME;
let uiData = {
  pendingPunch: null,    // 打刻確認中のデータ
  editingIndex: null,    // 編集中の打刻インデックス
  deletingIndex: null,   // 削除中のインデックス
  applyingType: null,    // 申請種別
  currentPage: 0         // 表示ページ
};
```

**影響度：** 🔴 **極大**  
→ 状態遷移図が定義できない  
→ 予期しない状態の組み合わせが発生しやすい  

**優先度：** 📍 ステップ5以降で構造的に改善推奨

---

### 2. 【State Sufficiency 違反】時刻管理が曖昧

**問題の本質：**
```javascript
let fakeTime = null;        // 固定時刻
let fakeTimeOffset = 0;     // リアル時刻との差分

// 3つの状態が可能
// 1. fakeTime = null, offset = 0        → リアル時刻
// 2. fakeTime = Date, offset = 0        → 固定時刻
// 3. fakeTime = null, offset ≠ 0        → リアル + 差分

// しかし以下も表現可能
// 4. fakeTime = Date, offset ≠ 0        → 矛盾状態！
```

**欠陥：** `fakeTime` と `fakeTimeOffset` が同時に非ゼロになることが防げない

例：
```javascript
// 「この時刻からリスタート」でうっかり設定忘れ
fakeTime = someDate;
fakeTimeOffset = 100;  // 両方非ゼロ → 状態が曖昧
```

**根治策：**
```javascript
const TimeMode = {
  REAL: "real",           // リアル時刻
  FIXED: "fixed",         // 固定時刻
  ADVANCING: "advancing"  // リアル + 差分
};

let timeMode = TimeMode.REAL;
let fixedTime = null;
let timeOffset = 0;

function getNow() {
  switch (timeMode) {
    case TimeMode.REAL:
      return new Date();
    case TimeMode.FIXED:
      return new Date(fixedTime);
    case TimeMode.ADVANCING:
      return new Date(new Date().getTime() + timeOffset);
  }
}
```

**影響度：** 🟡 **中程度**  
→ 開発者モードのみなので本番影響は低い  
→ しかしデバッグが複雑になる可能性

---

### 3. 【State Closure 違反】Firestoreとローカルの同期が脆い

**問題の本質：**

現在のフロー：
```
1. ユーザーが「出勤」をクリック
   ↓
2. ローカル history に追加
3. Firestoreに非同期保存
4. Firestoreリスナーが発火
5. history を Firestore データで上書き
```

**隠れた欠陥：**

```javascript
// 例：編集中にリスナーが発火
history[i].time = "10:00";  // 編集中
// → その時点でリスナー発火
// → Firestore データで上書き
// → 編集が失われる
```

**欠陥：** 「ローカルとFirestoreが完全に同期」という前提が脆い

- オフラインで編集したデータは失われる
- ネットワーク遅延中の操作は失われる
- 複数タブから同時編集すると衝突

**根治策：**
```javascript
// Firestoreデータは「単一の真実」として扱う
let lastSyncedData = null;  // Firestoreから取得した最新データ

// ローカル編集
let pendingChanges = {
  [docId]: { field: "time", value: "10:30" }
};

// リスナー発火時はマージする
// Firestore の変更 + ローカルの pending changes
```

**影響度：** 🔴 **大きい**  
→ データロスのリスク  
→ 複数デバイスでの競合リスク

---

## 🟡 中程度の欠陥（改善推奨）

### 4. 【Reachability 潜在的破壊】オーバーレイの重なり方が明示的でない

**現在のz-index構成：**
```css
#loginView:        z-index: 200
#overlay:          z-index: 50
#settingsPanel:    z-index: 90
#devPanel:         z-index: 100
```

**問題：** z-indexの階層が「なぜこの順番？」が明確でない

→ 将来的に新しいパネルを追加するときに重なり順がわからない

**改善案：**
```css
/* レイヤー定義を明示的に */
#background:       z-index: 1
#mainContent:      z-index: 10
#devPanel:         z-index: 100   /* 開発者専用 */
#settingsPanel:    z-index: 110   /* 設定（ユーザー向け） */
#overlay:          z-index: 120   /* ダイアログ背景 */
#loginView:        z-index: 200   /* 最前面 */
```

**影響度：** 🟡 **低→中**（スケール時に重要）

---

### 5. 【State Sufficiency 潜在的問題】`makeRow()` と `makeAuditItem()` の責務が曖昧

**問題：**
```javascript
// makeRow() は view層の関数だが、イベントリスナーで状態も変更している
editBtn.addEventListener("click", function () {
  openEdit(index);  // ← ここでUIState を変更
});

removeBtn.addEventListener("click", function () {
  item.canceled = true;  // ← ここでデータも変更
});
```

**欠陥：** View生成と状態管理が混在している

**根治策：**
```javascript
// View層：表示のみ責務
function makeRow(item, index) {
  // rows要素を作成して返す、イベントは登録しない
}

// Controller層：イベント処理
function setupRowEventListeners() {
  // rows のイベントをここで全て処理
}
```

**影響度：** 🟡 **中程度**  
→ テスト困難性が上がる  
→ コードの追跡が複雑になる

---

## 🟢 低優先度の欠陥（将来対応でOK）

### 6. ローカル編集（打刻修正）がFirestoreに反映されていない

**現在：**
```javascript
// 打刻を修正（timeInput で時刻を変更）
fillDone() {
  const newTime = timeInput.value + ":00";
  item.time = newTime;  // ← ローカルのみ
  item.pending = true;
  renderHistory();
}
```

**欠陥：** ローカルで修正したデータがFirestoreに保存されない

→ Firestoreリスナーで古いデータが戻ってくる可能性

**改善案：**
```javascript
fillDone() {
  const newTime = timeInput.value + ":00";
  item.time = newTime;
  item.pending = true;
  
  // Firestoreにも保存
  if (item.docId && isFirebaseConfigured) {
    db.collection("users").doc(currentUser.uid)
      .collection("punches").doc(item.docId)
      .update({ 
        time: newTime,
        edited: true
      });
  }
  
  renderHistory();
}
```

---

## 📊 欠陥レベルサマリー

| 項目 | 原則 | 深刻度 | 優先度 |
|------|------|--------|--------|
| UI状態の散在 | State Sufficiency | 🔴 極大 | 📍 高 |
| 時刻管理の曖昧さ | State Sufficiency | 🟡 中 | 📍 中 |
| Firestore同期の脆さ | State Closure | 🔴 大 | 📍 高 |
| z-index の不明確さ | Reachability | 🟡 低→中 | 📍 低 |
| 責務の混在 | 全般 | 🟡 中 | 📍 中 |
| 修正内容の非永続化 | State Sufficiency | 🟢 低 | 📍 低 |

---

## 🎯 改善ロードマップ

### Phase 1（緊急）：State Sufficiency の統一
```
ステップ5-6で実装するときに、UIStateを統一する。
→ pending, fillType, editIndex, deleteIndex, page を単一の状態変数に統合
```

### Phase 2（重要）：Firestore同期の信頼性向上
```
ステップ6-7で実装。
→ ローカル編集内容をFirestoreに永続化
→ オフライン対応を追加
```

### Phase 3（改善）：開発者体験の向上
```
次期リリースで実装。
→ TimeMode の明確化
→ View と Controller の分離
```

---

## 結論

**現在のコード品質：**
- 機能的には正常に動作している ✅
- しかし**状態管理が曖昧**で、将来の拡張時に問題が出やすい ⚠️

**推奨アクション：**
1. 今すぐ： Phase 1 の計画を立てる
2. ステップ5実装時： UIState を導入
3. ステップ6実装時： Firestore同期を強化

**メモ：** これはあくまで「設計上の改善提案」です。現在のコードは「動作している」ので、急ぐ必要はありませんが、拡張時に参考にすると良いでしょう。
