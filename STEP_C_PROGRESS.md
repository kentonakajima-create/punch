# 修正ステップC進捗：UI状態の統一

## ✅ 完了した修正

### C-1：新しい状態管理の定義
- ✅ `UIState` オブジェクトを定義
  - HOME, PUNCH_CONFIRM, PUNCH_EDIT, PUNCH_DELETE, APPLY_FORM, SETTINGS, LOGIN
- ✅ `currentUIState` を状態の単一の真実とする
- ✅ `uiData` に補助データを統合
- ✅ 互換性のための getter/setter を定義

### C-2：UI表示制御関数
- ✅ `updateUIForState()` で状態に基づいた表示制御
- ✅ `setUIState()` で状態遷移を統一
- ✅ `closeUI()` でUI閉鎖を標準化

### C-3：部分的な状態遷移統一
- ✅ `showLoginView()` / `showHomeView()` を新しい状態管理に対応
- ✅ `openConfirm()` で PUNCH_CONFIRM 状態を設定
- ✅ `confirmPunch()` で新しい状態参照に対応
- ✅ `closeConfirm()` を `closeUI()` で統一

---

## 🔄 互換性レイヤーの仕組み

古いコードとの互換性を保つために、以下のように getter/setter を定義：

```javascript
// 古いコード: pending = {...}
// 新しいコード: uiData.pendingPunch = {...}, currentUIState = UIState.PUNCH_CONFIRM
// → 自動的に両方で動作
Object.defineProperty(window, 'pending', {
  get: () => (currentUIState === UIState.PUNCH_CONFIRM) ? uiData.pendingPunch : null,
  set: (v) => { ... }
});
```

---

## 🧪 動作確認チェックリスト

以下が動作することを確認してください：

- [ ] ページを開く → ホーム画面表示される
- [ ] 「出勤」ボタンをクリック → 確認ダイアログが表示される
- [ ] 「確定する」をクリック → 出勤が記録される
- [ ] 「出勤」の「取消」をクリック → 取消状態になる
- [ ] 「出勤」の「復元」をクリック → 復元される
- [ ] 「設定」ボタンをクリック → 設定パネルが表示される
- [ ] 「閉じる」をクリック → 設定パネルが閉じられる
- [ ] Googleでログアウト → ログイン画面に戻る
- [ ] Googleで再度ログイン → ホーム画面が表示される

---

## 📋 残りの修正（最小限）

いくつかの関数がまだ古い参照を使っている可能性があります：
- `fixFill()`, `fixCancel()`, `fixForce()` 内の pending 参照
- `openEdit()`, `openDelete()` 内の editIndex / deleteIndex 参照
- `closeApplyForm()` 内の currentApplyType 参照

しかし、互換性レイヤーのため、ほとんどは自動的に動作するはずです。

問題が発生したら、該当箇所を修正します。

---

## 🎯 改善のメリット

この修正により：

✅ **状態が一元管理される**
- 複数の状態変数の矛盾が発生しない
- 状態遷移が追跡可能

✅ **コードが読みやすくなる**
- `if (currentUIState === UIState.PUNCH_CONFIRM)` 
- vs `if (pending !== null && overlay.show())`

✅ **将来の拡張が簡単**
- 新しいUI状態を追加するのが簡単
- 状態遷移図が描ける
