# 振替伝票アプリ - 開発まとめ（DEVLOG）

> **このファイルについて**: 開発の詳細な経緯・検討過程・不具合調査記録は、Obsidian（開発者PC/OneDrive内、
> リポジトリ外）ではなく**このリポジトリ内のDEVLOG.mdで管理する**。理由は、Obsidianノートは特定PC・
> 特定OneDriveアカウントに紐付いており`git clone`だけでは引き継げないため（genka-webappと同じ運用）。

対象ディレクトリ: `支払伝票PJ\transfer-slip-app\`
GitHubリポジトリ: https://github.com/shar19-ops/transfer-slip-app
対象ファイル: `振替伝票.html`（単一HTMLファイルで完結・外部ファイル依存なし。`date-stamp.js`のみ別ファイルだったが、d7771e3でインライン化済み）

---

## 2026-07-19: Obsidianとgitリポジトリの乖離発覚・統合

### 発覚の経緯

「振替伝票App改修（機能追加）」を進める前段階として、バージョン情報の新旧をObsidianとGitHubで確認したところ、
以下の乖離が判明した:

| ファイル | 場所 | 最終更新 | 借方貸方一致チェック機能 | 印鑑欄/役職/デート印/日本語ファイル名(d7771e3の変更) |
|---|---|---|---|---|
| `振替伝票.html` | 会社アカウント側OneDrive `OneDrive - 株式会社雄電社\デスクトップ\伝票印刷webアプリ版\`（**gitリポジトリ外**） | 2026-06-30 | ✅ あり | ❌ なし |
| `振替伝票v2.html` | 同上デスクトップフォルダ | 2026-07-06 | ❌ なし | ✅ あり（このリポジトリのHEADとバイト単位で完全一致） |
| `振替伝票.html`（本リポジトリ） | `transfer-slip-app`（個人OneDrive `MCフォルダ`配下） | 2026-07-06（d7771e3） | ❌ なし | ✅ あり |

原因: 2026-06-30に「借方貸方一致チェック機能」を追加した際の作業場所が、gitリポジトリ管理下の
`transfer-slip-app`フォルダではなく、会社アカウント側OneDriveの配布用デスクトップフォルダ
（`伝票印刷webアプリ版`）だった。Obsidianノート（2件、2026-06-30付）にはこの変更内容が記録されていたが、
コミットもpushもされていなかったため、その後gitリポジトリ側で進んだ改修（印鑑欄・役職・デート印機能、
日本語ファイル名化など、d7771e3）には反映されず、機能が「片方にしかない」状態のまま2つの系統に分岐していた。
支払伝票アプリ（一般管理費）で発覚したv2/v3リポジトリ乖離と同種の問題。

### 対応

デスクトップ版（06-30時点）から「借方貸方一致チェック機能」をこのリポジトリの`振替伝票.html`に移植し、統合。

- 控え選択モーダル（`#export-modal`）に警告用`<p id="export-mismatch-warn">`を追加（初期状態`display:none`）
- `openExportModal()`内で`updateTotal()`を呼び、返り値`{debitSum, creditSum}`が不一致なら警告を表示、
  一致なら非表示にする処理を追加
- 出力ボタン（`confirmExport()` → `generatePDF()`）側は変更なし＝警告が出ていても出力・キャンセルは自由に選択可能
  （出力をブロックしない仕様。デスクトップ版の実装をそのまま踏襲）

ローカルHTTPサーバー（`python -m http.server`）+ Playwrightで動作確認:
- 借方金額のみ入力（貸方0のまま）→ 印刷ボタン押下 → 警告メッセージが表示されることを確認
- 貸方金額を借方と同額に修正 → 再度印刷ボタン押下 → 警告が非表示になることを確認

コミット: `a489803`

### 今後のTODO

- [ ] 会社アカウント側OneDriveの配布用デスクトップフォルダ（`伝票印刷webアプリ版`）は今後gitリポジトリ外の
      「配布先」としてのみ使い、機能追加・修正は必ず本リポジトリ（`transfer-slip-app`）側で行う運用を徹底する
- [ ] デスクトップフォルダの古い`振替伝票.html`（06-30版、印鑑欄等の機能なし）の扱いを決める
      （本リポジトリのHEADで上書きするか、しばらく残すか）
- [ ] 科目・経費CD・収支項目の追加がlocalStorage限定でPC/ブラウザごとに独立している問題（社内共有不可、
      キャッシュクリアで消失）は未着手のまま。詳細はObsidianノート
      「振替伝票PJ - 科目･経費CD･収支項目データの保存先と今後の検討課題 (2026-06-30)」を参照

## 2026-07-19: 請求書PDF添付機能の追加

### 要件

請求書のPDFファイルを添付できるようにしたい。保存の際もそのPDFを取り込んでJSONファイルに含め、
JSONファイルを読み込むとPDFファイルも一緒に復元される仕組みにしたい、というユーザー要望。

### 実装

- `発行情報`カードの下に新規`▌ 添付ファイル（請求書PDF）`カードを追加
  - `📎 請求書PDFを添付`ボタン（実体は非表示の`<input type="file" accept="application/pdf,.pdf">`）
  - 添付中のファイル名表示、`👁 表示`（`window.open(dataUrl)`で新規タブ表示）、`🗑 削除`ボタン
- `attachedInvoicePdf`（`{ name, dataUrl } | null`）というグローバル変数で保持。`FileReader.readAsDataURL()`
  でPDFをbase64データURLに変換してメモリ上に保持する（サーバー送信なし、ブラウザ内完結）
- `collectFormData()`に`__invoicePdf: attachedInvoicePdf`を追加 → `saveData()`が呼ぶJSONにそのまま
  base64込みで埋め込まれる
- `applyFormData()`で`data.__invoicePdf`から`attachedInvoicePdf`を復元しUIを更新 → `loadData()`で
  JSONを読み込むとPDFも自動的に復元される
- `clearForm()`でも`attachedInvoicePdf`をリセットするよう追加

### 検証

ローカルHTTPサーバー + Playwrightで、ダミーPDFの添付→UI表示確認→
`collectFormData()`→JSON化→`removeInvoicePdf()`→`applyFormData(JSON.parse(...))`という一連の流れを
`page.evaluate()`で直接実行し、往復後にファイル名・表示ボタンが正しく復元されることを確認
（`window.showSaveFilePicker`/`showOpenFilePicker`はOSネイティブダイアログを伴うため自動化テスト対象外とし、
その手前のデータ層のみ検証した）。「🗑 削除」ボタンのクリックでも表示が「（未添付）」に戻ることを確認。

### 補足・今後の検討

- PDFはbase64化されるため元ファイルサイズの約1.33倍になり、JSONファイルのサイズが増える。今回は
  サイズ上限のバリデーションは未実装（要望になければ追加不要と判断）。
- 添付したPDFを印刷／PDF出力（`generatePDF()`）側にマージする機能は今回のスコープ外（要望されていない）。
  必要になれば`pdf-lib`（既にCDN読み込み済み）でページ結合する形が自然。

コミット: `15e3e0f`

## 2026-07-19: 請求書PDFの印刷／PDF出力へのマージ

### 要件

上記の添付機能に続けて、添付した請求書PDFを実際の印刷／PDF出力にもマージしたい、というユーザー要望。

### 実装

`generatePDF()`内、振替伝票のA4ページを`a4Doc`に全て追加し終えた直後（`a4Doc.save()`の手前）に、
`attachedInvoicePdf`があれば以下を実行:

- `dataUrl`のbase64部分を`atob`でデコードし`Uint8Array`化
- `PDFDocument.load()`で別の`PDFDocument`としてパース
- `a4Doc.copyPages(invoiceDoc, invoiceDoc.getPageIndices())` → `a4Doc.addPage()`で全ページを
  振替伝票ページの後ろに追記（元のページサイズのまま、A4/A5への強制リサイズはしない）

`withCopy`（控えあり/なし）は振替伝票側のA4ページ数自体を増やさない仕様（同じA4シートの上下に2面
描画するだけ）なので、請求書ページはcopy設定に関わらず1部だけ末尾に追加される。

結合失敗時（添付PDFが壊れている等）は`try/catch`で捕捉し、`invoiceMergeWarning`をstatus表示に追記
（`status-err`クラスで警告色）。ただし振替伝票本体の出力自体は失敗させず、印刷／PDF出力は必ず完了する
（既存の借方貸方不一致警告と同じ「警告するが止めない」方針を踏襲）。`printViaCanvas()`はpdf-lib結合後の
`outBytes`をpdfjsで丸ごとページ画像化する既存の仕組みをそのまま使うため、印刷モードでも追加コード不要で
請求書ページを含めて印刷できる。

### 検証

ローカルHTTPサーバー + Playwrightで、実PDF（`振替伝票(A5).pdf`、1ページ）をダミー請求書として添付し検証:
- `URL.createObjectURL`を差し替えてBlobを捕捉 → `generatePDF(false,'download')`実行 → pdfjsで
  ページ数を数え、振替伝票1ページ＋請求書1ページ＝**2ページ**になることを確認（警告メッセージなし）
- `window.print`を無害化した上で`generatePDF(true,'print')`実行 → `#print-area`に**2枚のcanvas**
  （振替伝票分＋請求書分）が生成されることを確認（`withCopy=true`でも請求書は1部のみ）
- `attachedInvoicePdf.dataUrl`を意図的に壊れたbase64にして再実行 → 振替伝票自体はダウンロード完了
  （`hasBlob:true`）しつつ、ステータスに「請求書PDFの結合に失敗しました: ...」という警告が
  `status-err`表示で追加されることを確認（本体出力をブロックしないことの検証）

コミット: `4fcdc5d`

## 2026-07-19: 不具合修正 - 添付PDFの「表示」ボタンで空タブしか開かない

### 症状

ユーザーがPDF添付機能を検証したところ、「👁 表示」ボタンを押すと新規タブは開くものの、中身が
何も表示されない（空白）状態になると報告。「前にも同じ事象があったと思う」とのコメントあり。

### 原因

`viewInvoicePdf()`が`attachedInvoicePdf.dataUrl`（`data:application/pdf;base64,...`形式）を
そのまま`window.open(url, '_blank')`に渡していた。Chromeは新規タブへの`data:` URLへの直接遷移を
セキュリティ上の理由でブロックする仕様があり、結果として空白タブが開くだけになっていた。

同じアプリ群の他アプリ（`一般管理費支払伝票アプリ_v3`の`支払伝票【一般管理費】.html`）を確認したところ、
ファイル保存など同種の場面では最初から`Blob` → `URL.createObjectURL()`で得た`blob:` URLを使っており、
`data:` URLを直接渡す実装は避けられていた。これが「前にも同じ事象があった」の実体と考えられる
（同種の落とし穴に別アプリでは既に対処済みだった）。

### 修正

`viewInvoicePdf()`内で、base64をデコードして`Uint8Array`化 → `Blob`化 → `URL.createObjectURL()`で
`blob:` URLを生成してから`window.open()`に渡すよう変更。他の関数（`saveData()`のフォールバック等）で
既に使われているパターンと統一。

### 検証

Playwrightで実PDFを添付→「表示」ボタンをクリック→開いた新規タブが`blob:`URLになっていることを確認
（`data:`URLではない）。そのタブのスクリーンショットを取得し、Chrome内蔵PDFビューアに振替伝票テンプレート
PDFの内容が正しく描画されていることを目視確認。

コミット: `d24cd52`

## 2026-07-19: ヘッダーにバージョン表示を追加

### 要件

ユーザー要望「ヘッダーの題名の脇にバージョン情報を記載させる」。

### 実装

genka-webapp（[[原価計算表アプリ - 開発まとめ (2026-07-11)]]、11章「ヘッダーにアプリのバージョン表示を
追加」）と同じ`v<日付><枝番アルファベット>`方式を採用。ただし振替伝票.htmlは単一HTMLファイルで外部
JS/CSSファイルへのキャッシュバスティング用`?v=YYYYMMDDx`クエリが元々存在しないため、genka-webappのように
既存の文字列を使い回すのではなく、この表示専用の文字列として新規に持たせた。

- `<h1>振替伝票 <span class="app-version">v20260719a</span></h1>`
- `.app-version { font-size:12px; font-weight:400; color:rgba(255,255,255,0.75); vertical-align:middle; margin-left:4px; }`
  （ヘッダーは濃い緑背景に白文字のため、白の半透明で「控えめ」を表現）
- 印刷時は既存の`body * { visibility:hidden }`（`#print-area`以外）仕様によりヘッダーごと非表示になるため、
  印刷物には出ない（追加対応不要だったことを確認済み）

### 運用ルール

このアプリを今後編集するセッションでは、`v20260719a`の日付・枝番（同日複数回なら a→b→c…）を
コード変更のたびに更新する（genka-webappの「教訓」と同じ：更新し忘れても実害は小さいが、
チェックリストに含めておく）。

### 検証

Playwrightでヘッダー部分のスクリーンショットを取得し、「振替伝票 v20260719a　入力フォーム」の
表示・レイアウトを目視確認。

コミット: `d18e463`

## 2026-07-19: PWAマニフェスト・アイコン追加（PC Chromeでインストールできない問題の解消）

### 症状

ユーザーから「PCブラウザーでPWA化ができない」と報告。

### 原因

このリポジトリには`manifest.json`もアイコンも一切存在しておらず、ChromeがPWAとして認識できる
情報自体が無かった（インストールボタン/メニューが出ようがない状態）。同じ支払伝票PJ系の
`一般管理費支払伝票アプリ_v3`では同日08:46頃に`icons/`＋`manifest.json`を追加済みだった
（[[支払伝票PJ - v2v3リポジトリ整理・配布方式・アイコン設定まとめ (2026-07-19)]]参照）のに対し、
`transfer-slip-app`は未対応のまま取り残されていた。

### 対応

`一般管理費支払伝票アプリ_v3`と同じ構成で追加:

- アイコン元画像: `F:\画像\DESKTOPアイコン\振替伝票.png`（392×392）から`Pillow`でリサイズし
  `icon-512.png` / `icon-192.png` / `apple-touch-icon.png`(180px) / `favicon-48.png` / `favicon-32.png`を生成
- `manifest.json`新規作成（`name`/`short_name`は「振替伝票」、`start_url: "./振替伝票.html"`、
  `theme_color`はヘッダー背景と同じ`#1a7a4a`）
- `<head>`にfavicon・apple-touch-icon・manifest・theme-colorのリンクタグを追加

**Service Workerはあえて実装しない**（`一般管理費支払伝票アプリ_v3`と同じ判断。pushすれば常に最新版を
表示させたい方針と、オフラインキャッシュによる「更新が反映されない」問題の再発リスクを避けるため）。

### 検証

Service Workerなしでも本当にインストール可能かどうかは、Chromeの`beforeinstallprompt`イベント発火を
自動テストで安定して検知するのが難しいため、Playwrightの`browser_run_code_unsafe`でCDPセッションを
直接開き、`Page.getInstallabilityErrors`を呼び出して確認する方法を採った。マニフェスト・アイコン追加後は
`{"installabilityErrors":[]}`（エラーなし＝インストール可能）を確認。マニフェスト・アイコンの存在だけで
Chromeの技術的なインストール可否判定は満たされ、Service Workerは必須要件ではないことが実証された。

コミット: `a4cf0ad`
