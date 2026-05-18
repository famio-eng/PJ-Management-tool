# Claude Code 指示文 — Phase 1b 改善実装

## 対象リポジトリ
https://github.com/famio-eng/PJ-Management-tool
対象ファイル：`Pj-Viewer.html`（単一ファイルにCSS・JS全内包）
※ファイル名を`index.html`から`Pj-Viewer.html`に変更すること（A-10参照）

---

## 前提
- `improvements/plan.md`・`improvements/progress.md`を先に読むこと
- Phase 1の実装は完了済み。残存バグ（Bug-A・Bug-B）は後回しにしてよい

---

## セッション分割方針

**Session A**（本ファイル）：全体共通・データ配信・ダッシュボード・タイムライン
**Session B**（plan_phase1b_sessionB.md）：NW図・ガントチャート・定例ビュー・機能別進捗報告

Session Aを動作確認してからSession Bに進むこと。

---

## システム構成・運用方針（重要・必ず把握すること）

### SharePoint上のフォルダ構成
```
SharePoint共有フォルダ
├── Pj-Viewer.html   ← アプリ本体（改修時のみPMが更新）
├── data.json        ← PMが毎週Excelから出力して上書きアップロード
└── archive/         ← 旧バージョンのHTMLバックアップ置き場
```

### データ読み込みの優先順位
1. 起動時に`./data.json`をfetchで自動読み込み（SharePoint環境）
2. fetchに失敗した場合：LocalStorageのデータにフォールバック（ローカル開発環境）
3. LocalStorageにもデータがない場合：Excelを手動アップロードするよう促す

### ユーザー区分
- **PM**：Excelを読み込み→data.jsonを出力→SharePointにアップロードする
- **メンバー**：`Pj-Viewer.html`を開くだけで最新データを閲覧できる（設定は自分のブラウザ内で変更可能）

---

## Session A

---

### A-1. 全体共通：表示項目の編集（設定画面）

**要件**：
- ナビゲーションバーの「⚙ 設定」ボタンから開くモーダルに「表示列」タブを設置する
- 対象列候補：`ID / WBS番号 / アウトラインレベル / タスク名 / Note / 期間 / 開始日 / 終了日 / 基準開始日 / 基準終了日 / 実績開始日 / 実績終了日 / 残存期間 / 先行タスク / 後続タスク / リソース名 / 達成率 / 状況 / 成果物 / クリティカル / アクティブ / 進捗状況`
- 各列名の横にチェックボックスを配置し、チェックON=表示・OFF=非表示とする
- デフォルトON列：`タスク名 / リソース名 / 開始日 / 終了日 / 達成率 / 状況 / 進捗状況`
- 設定はLocalStorage（キー：`pj_visible_columns`）に保存し、次回起動時に復元する
- 「適用」ボタン押下時に現在表示中のビューを即時再描画する
- モーダル内に「この設定はご自身のブラウザにのみ保存されます」と表示する

**実装上の注意**：
- 非表示列の`<th>`・`<td>`に`display:none`・`width:0`・`padding:0`・`border:0`をセットで適用する
- 再表示時はLocalStorage保存値またはデフォルト幅を復元する

---

### A-2. 全体共通：担当部門のカンマ区切り対応

**要件**：
- `リソース名`列の値に`,`が含まれる場合（例：`RA,Pj-M`）、複数部門として扱う
- フィルタードロップダウンの値一覧をカンマ分割して個別表示する
- フィルター適用時の判定もカンマ分割して比較する
```javascript
// フィルター生成時
const resources = (row['リソース名'] || '').split(',').map(s => s.trim()).filter(Boolean);
resources.forEach(r => values.add(r));

// フィルター判定時
const rowResources = (row['リソース名'] || '').split(',').map(s => s.trim());
return rowResources.some(r => selectedValues.includes(r));
```
- カンマ区切りの場合の色は「複数部門」色（`#808080`）を適用する

---

### A-3. 全体共通：表記ゆれ正規化（エイリアス設定）

**要件**：
- A-1の設定モーダルに「エイリアス」タブを追加する
- 「Excelの値」欄は`<datalist>`を使ったオートコンプリート入力にする
  - 現在読み込まれているExcelの`リソース名`列の全ユニーク値（カンマ分割後）を候補として表示する
  - 手入力も可能にする
- 設定はLocalStorage（キー：`pj_alias_map`）に保存する
- データ読み込み時にエイリアスマップで`リソース名`を置換してから処理する（カンマ区切りの各部門名に個別適用）

---

### A-4. データ配信：SharePoint JSON方式の実装

**背景**：
- `Pj-Viewer.html`と同じSharePointフォルダに`data.json`を置く
- メンバーはHTMLを開くだけで最新データが自動表示される
- PMは週次でExcelを読み込み→`data.json`を出力→SharePointに上書きアップロードする

**実装内容①：起動時のdata.json自動読み込み**
- `DOMContentLoaded`時に`./data.json`をfetchする
- 成功した場合：JSONをパースしてプロジェクト選択画面を表示する
- 失敗した場合：LocalStorageにフォールバックする
- LocalStorageにもデータがない場合：従来のExcel手動アップロード画面を表示する
```javascript
async function loadDataJson() {
  try {
    const res = await fetch('./data.json', { cache: 'no-cache' });
    if (!res.ok) throw new Error('not found');
    return await res.json();
  } catch {
    return null;
  }
}
```

**実装内容②：data.json出力機能（PM用）**
- ナビゲーションバーに「📤 JSON出力」ボタンを追加する
- クリックすると以下の構造のJSONを生成してファイル名`data.json`でダウンロードする
```json
{
  "version": 1,
  "exportedAt": "2026-05-18T10:00:00",
  "projects": [
    {
      "name": "Improved ITM",
      "tasks": [ ...ExcelパースデータのJSON... ]
    }
  ],
  "config": {
    "visibleColumns": [...],
    "aliasMap": {...},
    "departmentColors": {...}
  }
}
```
- `config`にはPMのLocalStorageに保存されている設定値を埋め込む

**実装内容③：起動時のプロジェクト選択画面**
- data.jsonまたはLocalStorageにプロジェクトが1件以上ある場合、起動時にプロジェクト選択モーダルを表示する
- 選択するとそのプロジェクトのデータを読み込んでメイン画面を表示する
- 「＋新規プロジェクトを追加（Excelをアップロード）」ボタンも設置する

**実装内容④：設定の読み込み優先順位**
1. LocalStorageに設定がある場合：LocalStorageを優先（メンバーの個別設定を尊重）
2. LocalStorageに設定がない場合：data.jsonの`config`をデフォルト値として使用する

**実装上の注意**：
- `file://`プロトコルではfetchがCORSエラーになるため、必ずLocalStorageへのフォールバックを実装すること
- `version`フィールドを含め、将来の構造変更時に移行処理を追加できるようにすること

---

### A-5. ダッシュボードビュー：列の並び替え

**要件**：
- 「遅延タスク一覧」「直近タスク一覧」テーブルの列ヘッダーをドラッグ＆ドロップで並び替えできるようにする
- 既存の`applyColReorder()`関数を流用する
- 並び順はLocalStorage（キー：`pj_column_order_dashboard`）に保存する

---

### A-6. ダッシュボードビュー：遅延タスクのフィルター・ソート

**要件**：
- 「遅延タスク一覧」テーブルの列ヘッダーにフィルターアイコン（▼）と昇順/降順ソートアイコン（↑↓）を追加する
- フィルターは既存の`applyColFilter()`関数を流用する
- ソートはヘッダークリックで昇順→降順→解除の3ステートをローテーションする
- ソート中の列ヘッダーに`▲`（昇順）または`▼`（降順）を表示する
- ソート後にテーブルのDOMを再構築して表示に反映する
- ソート状態はLocalStorageに保存しない（セッションごとにリセット）

---

### A-7. タイムラインビュー：タスク一覧の列幅調整・表示項目連動

**要件**：
- タイムライン左側のタスク一覧パネルの列幅をドラッグで調整できるようにする
- 既存の`applyColResize()`関数を流用する。保存キー：`pj_column_widths_timeline`
- A-1の表示列設定に連動させる（設定で非表示にした列はタイムラインのタスク一覧にも非表示にする）

---

### A-8. タイムラインビュー：サマリータスクの折りたたみ（多階層対応）

**要件**：
- 任意の階層のサマリータスクで折りたたみを可能にする
- 折りたたみ判定：クリックしたタスクのアウトラインレベルをNとし、それより下の行のうち`アウトラインレベル > N`の行をすべて非表示にする
- 折りたたんだ場合はD3のタイムラインバーも非表示にする（行とバーを同期する）
- 親サマリーを折りたたむと配下の子サマリーのトグル状態もリセット（展開状態に戻す）
- 折りたたみ状態はLocalStorage（キー：`pj_timeline_collapsed`）に保存し、次回起動時に復元する

---

### A-9. タイムラインビュー：縦幅拡張・スクロール改善

**要件**：
- `ResizeObserver`でナビゲーション・コントロールバー・インポートボタン領域の高さを動的に取得して`height: calc(100vh - Xpx)`を計算する
- タスク一覧パネル上でのホイールイベントを`addEventListener('wheel', handler)`でキャプチャし、カレンダーパネルの`scrollTop`を同期する（タスク一覧にカーソルがある状態でも縦スクロール可能にする）

---

### A-10. ファイル名変更

**要件**：
- `git mv index.html Pj-Viewer.html`でリネームしてコミットする
- HTML内の`<title>`タグを「Pj Viewer」に変更する
- READMEがあれば対象ファイル名を更新する
- この作業を**Session Aの最初**に実施すること

---

## 実装上の注意事項

1. data.jsonのfetchはSharePoint（HTTPS）環境前提。`file://`プロトコルでは必ずLocalStorageにフォールバックすること
2. JSON出力ボタンは将来的にPM専用UIに移動することを想定してコードを分離しておくこと
3. LocalStorageキーの命名は既存の`pj_`プレフィックスを維持すること
4. 既存の動作を壊さないこと。特にステータス計算ロジック・NW図レイアウトは変更しないこと
