# 実装進捗

## システム構成（確定）

```
SharePoint共有フォルダ
├── Pj-Viewer.html   ← アプリ本体（index.htmlからリネーム済み or 予定）
├── data.json        ← PMが毎週Excelから出力して上書きアップロード
└── archive/         ← 旧バージョンのHTMLバックアップ置き場
```

データ読み込み優先順位：data.json fetch → LocalStorageフォールバック → 手動アップロード

---

## 完了（動作確認済み）

### Phase 1 ビュー共通
- 1-1. 「報告日」→「次回定例予定日」ラベル変更
- 1-2. 列幅の手動ドラッグ調整
- 1-3. 列フィルター機能
- 1-4. 列の並び替え
- 1-5. ナイトモード
- 1-6. プロジェクト名編集UI削除

### Phase 1 大日程タイムライン
- 2-1. 実進捗バー重畳表示
- 2-2. 担当部門別バー色分け＋凡例
- 2-3. 「＋アイテム追加」ボタン非表示

### Phase 1 NW図
- 3-1. Closeタスクのグレーアウト
- 3-2. リソース名（担当機能名）表示
- 3-3. Phase 2プレースホルダー表示
- 3-4. サブグラフパネル展開（右クリック・コンテキストメニュー）

### Phase 1 ガントチャート
- 4-1. タイトル「ガントチャート（WBS）」変更
- 4-2. 実績バー重畳表示
- 4-3. 列ヘッダー項目名・2段ヘッダー
- 4-3b. 固定列＋カレンダー列の左右分割レイアウト（Bug-Aあり）
- 4-4. サブグラフアイコンクリック→新規ウィンドウ表示（Bug-Bあり）

### Phase 1 定例ビュー
- 5-1. コメント別管理（meeting / dept 分離）
- 5-2. 期限列削除

### Phase 1 機能別進捗報告
- 6-1. 「All」タブ追加
- 6-2. コメント同期
- 6-3. 送信完了トースト通知

### Excel読み込み改善
- 日付文字列パース（`2024年04月15日 8:00` / `2024/04/15` / `N/A`対応）
- 達成率の0〜1小数→0〜100整数変換
- シート名`タスク_テーブル1`対応

### Phase 1b Session A（完了）
- A-1. 表示列設定モーダル（⚙設定 > 表示列タブ）：完了
- A-2. 担当部門カンマ区切り対応：完了（getDeptColor・フィルター対応済み）
- A-3. エイリアス設定datalist対応（⚙設定 > エイリアスタブ）：完了
- A-4. data.json自動読み込み + プロジェクト選択モーダル + JSON出力ボタン：完了
- A-5. ダッシュボード列並び替え（applyColReorder）：完了
- A-6. ダッシュボード遅延タスク3ステートソート：完了
- A-7. タイムライン列幅調整（applyColResize/pj_column_widths_timeline）：完了
- A-8. タイムライン多階層折りたたみ（子サマリー連動・pj_timeline_collapsed保存）：完了
- A-9. ResizeObserver高さ動的計算 + ホイールスクロール同期：完了
- A-10. index.html → Pj-Viewer.html リネーム・titleタグ更新：完了

### Phase 1b Session B（完了）
- B-1. NW図タイムラインNWモード追加（renderTimelineNW・モード切替ボタン）：完了
- B-2. ガントチャート「全期間表示」ボタンを凡例エリア内に移動：完了
- B-3. ガントチャートタスク情報列最小幅保証（ensureGanttMinWidths）：完了
- B-4. ガントチャートTODAY列赤border-left・視認性向上：完了
- B-5. ガントチャートサマリータスク多階層折りたたみ（wbsプレフィックス方式）：完了
- B-6. ガントチャートID=1プロジェクトサマリー行表示切り替えチェックボックス：完了
- B-7. 定例ビュー textareaオートリサイズ・スクロールバースタイル統一：完了
- B-8. 機能別進捗報告 textareaオートリサイズ・saveDeptCommentにトースト：完了

---

## バグ修正済み

### [Bug-A] ガントチャート固定列がウィンドウ幅いっぱいに展開される → 修正済み
- `enhanceGanttTable()`冒頭で`localStorage.removeItem('pj_column_widths_gantt')`を実施

### [Bug-B] NW図サブグラフが新規ウィンドウで描画されない → 修正済み
- `buildSubgraphWindowHTML`をD3なしのバニラJS SVG実装に全面書き換え
- CDN依存・window.opener依存を完全排除

### [Bug-C] A-1 表示列設定がビューに反映されない → 修正済み
- 設定モーダルの「適用」ボタンでrenderScreen()を明示的に呼び出し
- pj_visible_columnsにLocalStorage保存・起動時復元

### [Bug-D] A-4 起動時プロジェクト選択画面が表示されない → 修正済み
- DOMContentLoadedでdata.json fetchを試みてプロジェクト選択モーダルを表示

### [Bug-D] A-4 起動時プロジェクト選択画面が表示されない
- `DOMContentLoaded`の先頭でdata.json fetchまたはLocalStorage確認を行い、データがある場合は選択モーダルを表示すること

---

## 未着手
（Phase 1b Session A / B の全項目が完了しました）

### Phase 2 以降（将来予定）
- Graph API連携（Azure AD ClientID取得後）
- B-8-4: 機能別進捗報告 全タスク表示モード（Close含む全期間表示）
- B-8-5: 機能別進捗報告 タスク情報/進捗コメント幅リサイズ

---

## 技術的負債・懸念事項

- ガントチャートの`enhanceGanttTable()`と`applyColResize()`でコード重複あり
- NW図サブグラフのD3インライン化が未実施
- ダークモードとD3描画エリアの色同期が一部未完全

## Phase 3 将来構想

### Power Automate × Teams 定例リマインド投稿
- Power DesktopでブラウザをキャプチャしTeamsに自動投稿
- ステータス：未実装・Phase 3以降で検討・Power Automateライセンス確認が前提

### Phase 2 Graph API連携
- ClientID取得後にSharePoint上のExcelを直接fetch・進捗コメントをSharePointに保存
- メンバーのコメント集約・リアルタイム同期が実現可能
- ステータス：IT部門へのAzureアプリ登録依頼待ち
