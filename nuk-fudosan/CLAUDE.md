# Property Note - 開発ガイド

## プロジェクト概要

不動産投資家・ゆうすけさんの個人用ツール。スマホメインで現場優先。
単一HTMLファイル（index.html）で、GitHub Pagesにホスティング。

- URL: `ayakanayuki923-coder.github.io/fudosan/nuk-fudosan/index.html`
- バックエンド: Google Apps Script + Googleスプレッドシート
- ローカル保存: localStorage キー `fudosan_note_v2`（**絶対変更禁止**）
- 完全仕様書: `property-note-spec-v2.md` を必ず参照すること

## コンセプト（最重要）

- **現場優先** — 外出先・手が塞がっている状況でも使える
- **スマホがメイン** — PCでも快適に使える
- **シンプル・高速・確実** — 商用ではないので過剰な機能は不要
- **会計ソフトは作らない** — 投資判断に必要な収支管理に徹する。確定申告・正式帳簿はfreee等に委譲。CSVエクスポートで橋渡し

## 技術構成

- 単一HTMLファイル（index.html）に全CSS/JSをインライン
- 外部ライブラリ不使用（Web標準APIのみ）
- レスポンシブ: PC（900px以上）はサイドバー、スマホはタブバー
- データはlocalStorageに保存、GAS連携で同期

## 開発ルール（絶対守る）

1. **アップロードされたファイルをベースに戻さない**
2. **修正JSは必ず `node --check` で構文確認してからHTMLに挿入**
3. **ファイルサイズが10KB以上減ったら異常として中断**
4. バージョン変更時は「Ver XX.X · Property Note」と「`<title>`」の2箇所を更新
5. 修正後は必ず主要機能の存在を確認（下記チェックリスト参照）
6. **localStorageキーは `fudosan_note_v2` から絶対に変更しない**

## 絶対に再発させないバグ

1. **id型不一致** — GASからのidは文字列、生成時も文字列
   → 常に `String(i.id) === String(id)` で比較
2. **renderFilterBars nullクラッシュ** — 非アクティブページのDOM要素
   → 必ずnullチェックしてから操作
3. **pullFromSheets後にrender忘れ**
   → pullFromSheets完了後に `renderCurrentPage()` を呼ぶ
4. **saveMemoがmemo-propertyプルダウンを参照**
   → `getSelectedMemoProperties()` を使う
5. **JS文字列内の実際の改行**
   → 文字列リテラル内は必ず `\n` でエスケープ
   → innerHTML内のonclickのクォートはテンプレートリテラルで回避

## ID生成ルール（全オブジェクト共通）

```javascript
id: String(Date.now()) + '-' + Math.random().toString(36).substr(2, 5)
// 例: "1712567890123-a3k9f"
```

Date.now()単体は音声入力等の連続操作で衝突リスクあり。全オブジェクト（物件/Memo/Learn/Journal/予定/cashflow/loan等）で統一。

## データ構造

```javascript
data = {
  property: [],      // 物件 → GASシート: 物件
  action: [],        // Memo → GASシート: Memo
  learning: [],      // 旧学習記録（後方互換） → GASシート: Note
  learns: [],        // Learn（新） → GASシート: Learn
  diary: [],         // Journal（旧名互換） → GASシート: Journal
  journals: [],      // Journal
  schedules: [],     // カレンダー予定 → GASシート: カレンダー予定
  gasUrl: '',
  gasApiKey: '',
  gasWritePassword: '',
  calcPatterns: [],   // 利回り電卓パターン
}
```

### 物件オブジェクト主要フィールド

- 基本情報: id, date, name, location, mapUrl, ptype, age, area, land, access, listingUrl, status
- 法的: zoning, coverage, far, road, rebuild, legalMemo
- 収支（見込み）: price, rentExpected, rentMarket, repairCost, closingCost, downPayment
- 融資: `loans: []`（配列・複数金融機関対応）, activeLoanId
- 出口: exitStrategy, exitTargetYear, exitTargetPrice, exitReason
- 見送り: dropReason, dropMemo, dropDate（status=見送り時のみ）
- 写真: photoAlbumUrl, photoMemo（Amazonフォト等の外部URL方式）
- 収支実績: `cashflows: []`（配列・1行ずつ記録）
- 取得確定値: actualPrice, actualClosingCost, actualDownPayment, purchaseDate
- その他: timeline, todos, richNote, memo

### ステータス（全ライフサイクル・順序固定）

```
取得: 候補発見 → 調査中 → 内覧予定 → 内覧済 → 交渉中 → 買付済 → 契約済
運用: 修繕中 → 募集中 → 運用中
出口: 売却検討中 → 売却済
離脱: 見送り
```

### Memoカテゴリ（12個・表示順固定）

```
物件探し / 現地調査 / 相場調査 / 法的確認 /
交渉・契約 / 融資 / 修繕・リフォーム /
入居・管理 / 収支・税務 / 売却 /
人脈づくり / その他
```

設計思想: 「何をした？」に対して迷わず1つ選べること。行動フェーズ順の並び。「業者」カテゴリは独立させない（業者は全カテゴリに絡むため目的別に吸収）。

### Learnカテゴリ

書籍 / セミナー / 現場 / ネット / YouTube / その他

### 融資オブジェクト（物件.loans配列内）

```javascript
{
  id, bank, amount, rate, years, type('元利均等'|'元金均等'),
  monthly, status('未相談'〜'承認'/'否決'), memo, result,
  archived: false, archivedDate: ''
}
```

否決・見送り時はarchived=trueで非表示。UIでは「過去の打診履歴」として折りたたみ表示。

### 収支（cashflow）オブジェクト

```javascript
{
  id, date, type('income'|'expense'), category, accountTag(''),
  amount, memo, recurring: false
}
```

- accountTag: 勘定科目タグ。任意入力・後付け可。確定申告時に必要に応じて埋める
- recurring: 毎月定額フラグ
- CSVエクスポート機能あり（BOM付きCSV、freee等対応）

## UI構成

### ページ一覧
- Home（ダッシュボード）: カレンダー折りたたみ、ToDo、物件、Memo、Learn
- 物件: 一覧（フェーズフィルタ）→ 詳細（Overview/Log/融資/収支/Noteタブ）
- Memo: 一覧（カテゴリフィルタ）
- Learn: 一覧（カテゴリフィルタ）
- カレンダー: 月表示、日付タップで全記録表示、予定・日記
- Settings: GAS接続設定

### タブバー（スマホ・5タブ）
Home / 物件 / Memo / Learn / カレンダー

### 共通UI仕様
- 日付入力: `type="text"` placeholder="YYYY-MM-DD"、モーダル開時に今日を自動セット
- 保存成功: ✅ 緑トースト 2.5秒
- 保存失敗: ❌ 赤トースト
- スマホ: padding-bottom: 150px !important
- overscroll-behavior: contain（pull-to-refresh無効）
- モーダル: 下からスライドアップ（スマホ）、中央表示（PC）

## 設計判断の記録

| 項目 | 決定 | 理由 |
|------|------|------|
| 写真保存方式 | 外部アルバムURL + メモ | localStorage 5MB制限。Amazonフォトで運用 |
| ID生成 | Date.now() + ランダム文字列 | 連続操作でミリ秒衝突を防止 |
| 会計機能 | 投資判断用の収支管理のみ | BS/PL・正式帳簿はfreee等に委譲 |
| accountTag | 任意入力・後付け可 | 最初は空欄でOK。確定申告時に必要に応じて埋める |
| Memoカテゴリ | 12個フラット表示 | グループ分けUIは不要。行動フェーズ順の並びで直感的に選択可能 |
| ステータス | 全ライフサイクル14段階 | 候補発見〜売却済まで1物件を1レコードで追い続ける |
| 融資情報 | 複数金融機関対応（配列）+アーカイブ | 複数打診が基本。否決理由の蓄積が次の打診戦略になる |
| 収支管理のツール分担 | Property Note+スプレッドシート+freee | 記録の入口はProperty Note、月次集計はスプレッドシート、確定申告はfreee等 |
| 利回り電卓と収支タブ | 共存（役割分離） | 電卓=購入前シミュレーション、収支タブ=購入後の実績記録 |

## 修正後チェックリスト

以下の機能・要素が存在することを確認する:

- [ ] showPage() で全ページ遷移が動作する
- [ ] openPropertyModal / saveProperty が動作する
- [ ] openMemoModal / saveMemo が動作する（getSelectedMemoProperties使用）
- [ ] openLearnModal / saveLearn が動作する
- [ ] openCalcModal / calcResult が動作する（パターン追加・削除）
- [ ] openLoanModal / saveLoan / archiveLoan が動作する
- [ ] openCashflowModal / saveCashflow / exportCashflowCSV が動作する
- [ ] カレンダー表示・予定追加・日記追加が動作する
- [ ] ToDo表示・チェックが動作する
- [ ] 音声入力フロー（openVoiceInput）が動作する
- [ ] GAS設定（testAndSaveGAS / syncDataNow）が動作する
- [ ] toast() が全保存操作で呼ばれている
- [ ] loadData / saveData でlocalStorageが正しく動作する
- [ ] スマホでタブバーが表示され、padding-bottomが150px
- [ ] 物件詳細の5タブ（Overview/Log/融資/収支/Note）が全て表示される

## ファイル構成

```
nuk-fudosan/
├── index.html                  ← メインアプリ（単一ファイル）
├── property-note-spec-v2.md    ← 完全仕様書
├── watch.html                  ← 既存のwatch（変更監視用）
├── index-old-backup.html       ← v1バックアップ
└── CLAUDE.md                   ← このファイル
```
