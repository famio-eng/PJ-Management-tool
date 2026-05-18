# 外部脳：PJ-Management-tool 開発ナレッジ

> 対象リポジトリ: `famio-eng/PJ-Management-tool`  
> 対象ブランチ: `claude/phase-1b-viewer-Dpi6q`  
> 最終更新: 2026-05-18

---

## アプリ概要

SharePoint上に配置する単一ファイルWebアプリ。CSS・JS全内包。

```
SharePoint共有フォルダ/
├── Pj-Viewer.html   ← アプリ本体（4195行）
├── data.json        ← PMが毎週Excelから出力して上書きアップロード
└── archive/         ← 旧バージョンHTMLバックアップ
```

**利用技術：**
- D3.js v7（CDN）：タイムライン・NW図SVG描画
- SheetJS / xlsx（CDN）：Excelインポート
- LocalStorage：全設定・データ永続化（キープレフィックス `pj_`）
- Vanilla JS SVG DOM API：NWサブグラフポップアップ（CDN非依存）

**ユーザー区分：**
- PM：Excel読み込み → `data.json`出力 → SharePointアップロード
- メンバー：`Pj-Viewer.html`を開くだけで最新データ閲覧

---

## ビュー構成

| ビュー | renderXxx() | 主な機能 |
|--------|-------------|---------|
| ダッシュボード | `renderDashboard()` | 遅延/直近タスク一覧、列並び替え・3ステートソート |
| タイムライン | `renderTimeline()` | ガントバー、多階層折りたたみ、列幅調整 |
| NW図 | `renderNW()` / `renderTimelineNW()` | D3トポロジカル / タイムライン配置 切替 |
| ガントチャート | `renderGantt()` / `drawGantt()` | WBS多階層折りたたみ、TODAY列強調、固定列分割 |
| 定例ビュー | `renderMeeting()` | meeting/dept コメント分離、textareaオートリサイズ |
| 機能別進捗報告 | `renderDept()` | タブ別表示、Allタブ、コメント同期、トースト |

---

## 重要な実装パターン

### データフロー（init()）

```javascript
async function init() {
  loadStorage();
  initTheme();
  const dataJson = await loadDataJson();   // HTTPSのみ、file://はnull即返し
  if (dataJson?.projects?.length) { /* 選択モーダル */ return; }
  const saved = loadProjectsFromStorage(); // pj_projects
  if (saved?.length) { /* 選択モーダル or 自動ロード */ return; }
  /* サンプルデータでダッシュボード表示 */
}
```

### LocalStorageキー一覧

| キー | 内容 |
|------|------|
| `pj_projects` | [{name, tasks}] — 保存済みプロジェクト全件 |
| `pj_visible_columns` | 表示列設定（文字列配列） |
| `pj_alias_map` | エイリアスマップ {Excel値: 正規名} |
| `pj_column_order_dashboard` | ダッシュボード列順 |
| `pj_column_widths_timeline` | タイムライン列幅 |
| `pj_timeline_collapsed` | タイムライン折りたたみ状態 |
| `pj_gantt_collapsed` | ガントチャート折りたたみ状態 |
| `pj_show_root_task` | ID=1サマリー行表示フラグ |
| `pj_nw_mode` | NW図モード ('dependency' / 'timeline') |
| `pj_nw_px_per_day` | タイムラインNW ズーム（px/日） |
| `pj_dept_show_all` | 機能別進捗 全タスク表示フラグ |
| `pj_theme` | テーマ ('light' / 'dark') |

### NWサブグラフポップアップ（CDN非依存化済み）

- `window.open()` + `document.write()` でBlobページを表示
- D3不使用。Vanilla JS SVG DOM API（`document.createElementNS`）で描画
- pan/zoom：mousedown/mousemove/wheel イベントで `transform` 属性を直接更新
- 関数：`buildSubgraphWindowHTML(dataJson)` → HTML文字列 → `newWin.document.write()`

### ガントチャート多階層折りたたみ（WBSプレフィックス方式）

```javascript
// 子孫判定：wbs が "1.2" のタスクを折りたたむと "1.2.x" 系が非表示に
state.tasks.forEach(t => {
  if (t.isSummary && t.wbs?.startsWith(task.wbs + '.')) {
    state.collapsedRows.delete(t.id); // 子サマリーのトグルをリセット
  }
});
```

### タイムラインNWモード（renderTimelineNW）

- X座標：`基準開始日` → `pxPerDay × 経過日数`
- Y座標：`アウトラインレベル` × 行高さ、同一行衝突時はY方向にずらす
- ノード幅：`期間(日) × pxPerDay`（最小120px）
- グリッド：月単位縦線（opacity:0.3）＋TODAY赤線
- D3 zoom/pan 有効（タイムラインNWモードでも）

---

## 未解決バグ

### [Bug-D] 起動時プロジェクト選択画面が表示されない

**環境：** `file://` プロトコル（ローカルファイルとして開いた場合）

**根本原因（2つ）：**

**①** `init()` で `savedProjects.length === 1` のとき `showProjectSelectModal()` を呼ばずに `selectProject(0)` を直接呼ぶ設計  
→ 1プロジェクトしかない場合は常にダッシュボードが直接表示される

**②** `pj_projects` に保存した tasks の日付フィールド（`bstart`/`bend`/`start`/`end`/`astart`/`aend`）が `JSON.stringify` で ISO文字列化され、`loadProjectsFromStorage()` 経由で読み込む際に `Date` オブジェクトへ戻す処理がない  
→ `selectProject()` 後の描画でDate演算が失敗する可能性

**修正コード（未適用）：**

```javascript
// 1. loadProjectsFromStorage() に日付再水和を追加
function loadProjectsFromStorage() {
  try {
    const s = localStorage.getItem('pj_projects');
    if (!s) return null;
    const projects = JSON.parse(s);
    const DATE_KEYS = ['bstart','bend','start','end','astart','aend'];
    projects.forEach(proj =>
      (proj.tasks || []).forEach(t =>
        DATE_KEYS.forEach(k => { if (t[k]) t[k] = new Date(t[k]); })
      )
    );
    return projects;
  } catch(_) { return null; }
}

// 2. init() — 件数によらず常にモーダル表示
const savedProjects = loadProjectsFromStorage();
if (savedProjects?.length) {
  _allProjects = savedProjects;
  showProjectSelectModal(savedProjects); // ← selectProject(0) ではなく常にモーダル
  return;
}
```

---

## 完了済み機能（Phase 1 〜 Phase 1b）

### Phase 1（全ビュー共通）
- 次回定例予定日ラベル変更、列幅ドラッグ、列フィルター、列並び替え、ナイトモード、プロジェクト名編集UI削除

### Phase 1b Session A
- A-1: 表示列設定モーダル（⚙設定 > 表示列タブ・`pj_visible_columns`）
- A-2: 担当部門カンマ区切り対応（`RA,Pj-M` → 複数部門色）
- A-3: エイリアス設定（datalist オートコンプリート・`pj_alias_map`）
- A-4: data.json自動読み込み＋プロジェクト選択モーダル＋JSON出力ボタン
- A-5: ダッシュボード列並び替え（applyColReorder）
- A-6: ダッシュボード遅延タスク3ステートソート
- A-7: タイムライン列幅調整（`pj_column_widths_timeline`）
- A-8: タイムライン多階層折りたたみ（`pj_timeline_collapsed`）
- A-9: ResizeObserver高さ動的計算＋ホイールスクロール同期
- A-10: `index.html` → `Pj-Viewer.html` リネーム

### Phase 1b Session B
- B-1: NW図タイムラインNWモード（renderTimelineNW・モード切替ボタン・`pj_nw_mode`）
- B-2: ガントチャート「全期間表示」ボタンを凡例エリア内に移動
- B-3: ガントチャートタスク情報列最小幅保証（ensureGanttMinWidths）
- B-4: ガントチャートTODAY列赤border-left（`#E53935`）・視認性向上
- B-5: ガントチャートサマリータスク多階層折りたたみ（wbsプレフィックス方式）
- B-6: ガントチャートID=1プロジェクトサマリー行表示切り替えチェックボックス
- B-7: 定例ビュー textareaオートリサイズ・スクロールバースタイル統一
- B-8: 機能別進捗報告 textareaオートリサイズ・トースト通知

### バグ修正済み
- Bug-A: ガントチャート固定列がウィンドウ幅いっぱいに展開 → `localStorage.removeItem('pj_column_widths_gantt')`
- Bug-B: NWサブグラフ新規ウィンドウで描画されない → Vanilla JS SVG に全面書き換え
- Bug-C: 表示列設定がビューに反映されない → `applySettings()` で `renderScreen()` 呼び出し

---

## 技術的負債

| 項目 | 内容 |
|------|------|
| コード重複 | `enhanceGanttTable()` と `applyColResize()` で列幅ロジック重複 |
| ダークモード | D3描画エリアの色同期が一部未完全 |
| localStorage容量 | `pj_projects` にtasks全件保存 → 大規模PJで5MB超えリスク |
| 日付型ロス | localStorage保存時にDate→文字列、読み込み時に再水和が必要（Bug-Dの根本） |

---

## 次回セッションでやること

1. **Bug-D修正**（優先度：高）
   - `loadProjectsFromStorage()` に日付再水和を追加
   - `init()` の1件スキップを廃止し常にモーダル表示

2. **動作確認依頼事項**
   - B-1 タイムラインNWモードの描画・zoom/pan動作
   - B-5 ガントチャート多階層折りたたみ
   - A-4 JSON出力→SharePointアップロード→メンバー閲覧フロー

3. **Phase 2（将来）**
   - B-8-4: 機能別進捗報告 全タスク表示モード
   - B-8-5: タスク情報/進捗コメント幅リサイズ
   - Graph API連携（Azure AD ClientID取得後）
