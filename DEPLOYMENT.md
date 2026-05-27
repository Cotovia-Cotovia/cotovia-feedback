# Cotovia Clinic フィードバックシステム — 構築・運用・トラブル対応ドキュメント

最終更新: 2026-05-27

---

## 1. システム全体像

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ GitHub Pages    │     │ Google Apps      │     │ Google Sheets   │
│ (フロント)       │ ──▶ │ Script (バック)   │ ──▶ │ (DB)            │
│  静的HTML×3     │     │  doPost受信       │     │  3シート         │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        ▲
        │
   ┌────┴─────┬──────────┐
   │          │          │
Zoom連携    UE院QR     WC院QR
(/)         (/ue/)     (/wc/)
```

### 構成要素
| 要素 | 役割 | 場所 |
|---|---|---|
| GitHub リポジトリ | フロントエンドのソース | https://github.com/Cotovia-Cotovia/cotovia-feedback |
| GitHub Pages | フォームページのホスティング（無料・HTTPS） | https://cotovia-cotovia.github.io/cotovia-feedback/ |
| Google Apps Script | POST受信 → Sheets書き込み | https://script.google.com/ |
| Google Sheets | 回答データの保存 | SHEET_ID `1tZFFm72l-yxNsllR048ylpA5rsSFx_-IDELKZuS9h1U` |

### 公開URL一覧
| 用途 | URL |
|---|---|
| オンライン診療（Zoom連携） | `https://cotovia-cotovia.github.io/cotovia-feedback/` |
| UE院 店内QR | `https://cotovia-cotovia.github.io/cotovia-feedback/ue/` |
| WC院 店内QR | `https://cotovia-cotovia.github.io/cotovia-feedback/wc/` |

---

## 2. データスキーマ

各シート（`online_feedback` / `ue_feedback` / `wc_feedback`）は共通スキーマ：

| 列 | キー | 内容 | 主な使い道 |
|---|---|---|---|
| A | 日時 | 送信時刻（JST） | 時系列分析 |
| B | 評価（1-5） | 星の数 | 平均満足度 |
| C | ご意見・ご感想 | 自由記述 | 内容分析 |
| D | デバイス情報 | UAヘッダー（ブラウザ/OS） | トラブル切り分け |
| E | 端末ID | localStorage の UUID | 同一端末の連投検知 |
| F | ミーティング情報 | Zoom クエリパラメータ（meetingId, userNameなど） | 該当ミーティング特定 |
| G | リファラー | document.referrer | Zoom経由か直接URLか判別 |

UE/WC シートでは F列・G列は通常空欄（QRスキャンには Zoom 情報がないため）。

---

## 3. 別クリニックでの流用手順（テンプレート）

クリニック名を `XYZ` とした例。所要時間: 約45分。

### Step 1: スプレッドシート作成（5分）
1. https://sheets.google.com/ で新規スプレッドシート作成
2. 名前を「XYZ Clinic Feedback」に変更
3. URL の `/d/` と `/edit` の間が **SHEET_ID** → メモ
4. シートを3つ作成: `online_feedback`, `<location1>_feedback`, `<location2>_feedback`

### Step 2: GAS プロジェクト作成（10分）
1. https://script.google.com/ → 新しいプロジェクト
2. 以下のコードを貼り付け（`SHEET_ID` と `SHEET_MAP` をクリニック用に変更）：

```javascript
const SHEET_ID = "YOUR_SHEET_ID_HERE";
const SHEET_MAP = {
  "online": "online_feedback",
  "location1": "location1_feedback",
  "location2": "location2_feedback"
};
const HEADERS = ["日時", "評価（1-5）", "ご意見・ご感想", "デバイス情報", "端末ID", "ミーティング情報", "リファラー"];

function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const source = data.source || "online";
  const sheetName = SHEET_MAP[source] || "unknown_feedback";
  const ss = SpreadsheetApp.openById(SHEET_ID);
  let sheet = ss.getSheetByName(sheetName);
  if (!sheet) { sheet = ss.insertSheet(sheetName); }
  const currentRow1 = sheet.getRange(1, 1, 1, HEADERS.length).getValues()[0];
  const headersMatch = currentRow1.every((cell, i) => cell === HEADERS[i]);
  if (!headersMatch) {
    sheet.getRange(1, 1, 1, HEADERS.length).setValues([HEADERS]);
  }
  sheet.appendRow([
    data.timestamp || "", data.rating || "", data.comment || "",
    data.userAgent || "", data.deviceId || "",
    data.meetingInfo || "", data.referrer || ""
  ]);
  return ContentService.createTextOutput(JSON.stringify({ status: "ok" }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

3. 保存（Ctrl+S）
4. デプロイ → 新しいデプロイ → ウェブアプリ
5. 設定:
   - 実行ユーザー: **自分**
   - アクセスできるユーザー: **全員**
6. 権限承認 → デプロイ → **Web アプリ URL** をコピー（`https://script.google.com/macros/s/.../exec`）

### Step 3: HTML 準備（5分）
1. このリポジトリの `index.html`, `ue/index.html`, `wc/index.html` をコピー
2. 各ファイル内の `GAS_URL` を新しいGAS URLに書き換え
3. 各ファイル内の `source` 値を新拠点名に変更（例: `"ue"` → `"location1"`）
4. タイトル（`<title>`）と本文の `<h1>` をクリニック名に変更

### Step 4: GitHub リポジトリ作成 & デプロイ（10分）
1. PowerShell で：
   ```powershell
   cd "C:\path\to\xyz-feedback"
   git init
   git config --local user.name "XYZ Clinic"
   git config --local user.email "admin@xyzclinic.com"
   git add .
   git commit -m "Initial commit"
   gh repo create xyz-feedback --public --source=. --remote=origin --push
   gh api -X POST "repos/<GH_USER>/xyz-feedback/pages" -f "source[branch]=main" -f "source[path]=/"
   ```
2. 3〜5分後に `https://<GH_USER>.github.io/xyz-feedback/` で公開確認

### Step 5: QR コード生成 & ポスター PDF（10分）
PowerShell の自動生成スクリプト（このリポジトリの `scripts/generate-posters.ps1` 参照）を流用。
変数 `$LOCS`、`$BASE`、`$BASE_URL` を新クリニック用に変更して実行。

### Step 6: 動作確認（5分）
- 各URLをブラウザで開き、星を選んで送信
- Sheets の各シートに行が追加されることを確認
- 同じブラウザから2回送信し、「端末ID」列に同じUUIDが入ることを確認

---

## 4. Zoom 運用方針（重要・2026-05-27 確定）

### 4.0 このシステムの位置づけ（前提）
**「ポジティブフィードバックの収集装置」** として運用する。

- ✅ 良い患者さんの声を収集してテスティモニアル・改善材料として活用
- ✅ スタッフのモチベーション維持
- ❌ **客観的な顧客満足度の測定ツールとしては使わない**
- ❌ 集まる評価は selection bias（医師の Survey 設定判断に依存）があるため、対外公表時は「平均X星」ではなく具体的なコメント引用で使う

この前提により、医師が Survey URL を毎回設定しなくても良い、受付やWebhookによる強制送信メカニズムは不要、という運用に落とし込む。

### 4.1 採用するアプローチ
**3rd-party survey link（会議終了時の自動オープン方式）** を継続使用する。

理由：
- 患者の行動コストが最小（ブラウザが自動で開く）
- 想定回答率 70-90%（他方式の数倍）
- 既存のカスタムページ・Sheets・端末ID連投検知などの仕掛けをフル活用できる
- 1日10件規模なら手動設定コストは合計100秒/日で許容範囲

### 4.2 不採用にした方式と理由
| 方式 | 不採用理由 |
|---|---|
| QR背景のみで運用 | 患者がスマホでスキャンする必要があり回答率が大幅に低下（5-15%想定） |
| Salepager等のZoom Marketplaceアプリ | 患者の Zoom Registration が前提となり、予約フロー全体の変更が必要 |
| Zapier / Webhook 自動メール | 患者メアドの一元取得経路がない（現状） |
| Zoom内蔵Survey | カスタムページ・Sheets連携・端末IDなど既存機能を全部失う |

### 4.3 アカウント側設定（一度だけ）
Account Owner / Admin 権限が必要。
1. https://zoom.us/ にログイン
2. **Account Management** → **Account Settings** → **Meeting** タブ
3. **Meeting Survey** セクション：
   - メイントグル → **ON**
   - **Allow host to use a 3rd-party survey link** → ✓ チェック
   - **Add specified participant survey to all meetings** → ✗ チェック外す（これは Zoom 内蔵 Survey用）
   - **Who can participate** → **External users only**
   - **Exclude hosts and co-hosts from taking survey** → ✓ チェック
4. **Save**

### 4.4 各ミーティングでの設定（毎回 10 秒）
テンプレートに Survey URL は引き継がれないので、毎ミーティング個別設定が必要：

1. ミーティング作成（テンプレ使用OK）
2. ミーティング詳細ページ → **Survey** タブ
3. **「Use a 3rd party survey」** ボタン
4. URL 入力欄に以下を貼付：
   ```
   https://cotovia-cotovia.github.io/cotovia-feedback/
   ```
5. **Save**

**コピペを楽にする小ワザ**：
- ブラウザのブックマークバーに「Survey URL」というブックマークを作り、URL を上記に設定 → 毎回クリックで取得
- Windows PowerToys / PhraseExpress でショートカット展開（例: `;feedback` → URL に展開）

### 4.5 Zoom から自動で付くクエリパラメータ
オンライン用 `index.html` は `window.location.search` を取得して `meetingInfo` 列に保存。
通常付与されるキー: `meetingId`, `meetingUuid`, `userName`, `userId` 等（Zoomバージョンにより変動）。

### 4.6 廃止アナウンスへの備え
Zoom 設定画面に「Hosts will no longer be able to use 3rd-party survey links...」というアナウンスが出ている。
- 正確な廃止日付は不明だが、6ヶ月〜1年で機能停止する可能性
- 廃止時の代替手段は別途検討する（Zoom Registration + Webhook → Email など）
- それまでは現運用を継続

### 4.7 補助的な施策（任意）
- `cotovia_zoom_background.png`: QRコード入り Zoom 仮想背景。Survey 設定漏れ時のフォールバック用。**メイン導線ではない**ため利用は任意。

---

## 5. Google Sheets / GAS の容量・制限

### スプレッドシート
| 項目 | 制限 |
|---|---|
| セル総数 | **1000万セル/spreadsheet**（7列なら約140万行） |
| 1シートのセル | 1000万まで |
| 同時編集者 | 100人 |

### Apps Script Web App
| 項目 | 制限（無料アカウント） |
|---|---|
| 1日の URL Fetch | 20,000 回 |
| 1日の実行時間合計 | 90 分 |
| 1回の実行 | 最大 6 分 |
| 同時実行 | 30 |

### 想定運用との比較
| 想定 | データ蓄積 | 制限到達まで |
|---|---|---|
| **1日10件** | 3,650/年 | 約 380 年で行制限 |
| 1日100件 | 36,500/年 | 約 38 年 |
| 1日1000件 | 365,000/年 | 約 3.8 年 |

**結論**: 1日10件レベルなら問題なし。クリニックの臨床ボリュームで上限に当たる可能性は実質ゼロ。

---

## 6. トラブルシューティング

### 「Sheets に記録されない」
| 確認項目 | 解決 |
|---|---|
| GAS URL が HTML に正しく入っているか | `index.html` を grep して URL 確認 |
| GAS の再デプロイ済みか | デプロイを管理 → 既存デプロイの新バージョン |
| GAS の権限承認済みか | GAS エディタで `doPost` を手動実行して承認 |
| SHEET_ID が正しいか | スプレッドシートのURL `/d/XXX/edit` の XXX 部分 |
| シート名がGAS設定と一致 | `SHEET_MAP` の値とSheets タブ名を見比べる |

### 「端末ID が入らない」
| 原因 | 解決 |
|---|---|
| ブラウザのHTMLキャッシュ | Ctrl+Shift+R（Win）/ Cmd+Shift+R（Mac）でハードリロード |
| GAS が古いバージョン | デプロイを管理から新バージョン作成 |
| `localStorage` がブロックされている | プライベートブラウズモードの可能性。通常モードで再テスト |

確認方法: フォームページで F12 → Console → `localStorage.getItem('cotovia_device_id')` 実行

### 「ミーティング情報 が入らない（Zoom経由なのに）」
| 原因 | 解決 |
|---|---|
| Zoom 側で 3rd-party survey 設定が ON になっていない | Zoom Account Settings → Meeting Survey 確認 |
| ホスト側 Meeting で「end-of-meeting feedback survey」が OFF | ミーティング設定で ON にする |
| 患者がアンケートをスキップ | これは仕様、対応不可 |

### 「PDF が見切れる / 1ページに収まらない」
| 原因 | 解決 |
|---|---|
| 印刷時の倍率が 100% | 印刷ダイアログで「**ページに合わせる / Fit to page**」を有効化 |
| PDF 内部に複数ページがある | `cotovia_qr_v3.html` 内の `page-break-after: always` を `auto` に変更 |
| デザイン自体が用紙より大きい | 自然寸法のPDF（210×330mm 等）を出して fit-to-page で印刷 |

### 「GitHub Pages の URL が 404」
- push 後の Pages ビルドに 1〜3 分かかる
- `gh api "repos/OWNER/REPO/pages/builds/latest" --jq '.status'` で確認
- `status: built` になるまで待つ
- ビルドエラー時は `.error.message` を確認

### 「Pages の更新が反映されない」
- ブラウザキャッシュの可能性。シークレットウィンドウで開いて確認
- Cloudflare/CDN キャッシュは GitHub Pages は約 10 分

---

## 7. メンテナンスチェックリスト

### 月次
- [ ] Sheets の行数を確認（10万行超えてきたら年次でアーカイブ検討）
- [ ] スパム送信や明らかな自動入力（同一端末ID連投）の有無を確認
- [ ] 平均評価と前月比較

### 年次
- [ ] GAS のデプロイバージョンを最新化（セキュリティパッチ反映のため）
- [ ] GitHub Pages の証明書/ドメイン確認
- [ ] GAS の URL が変わっていないか確認

---

## 8. 連投検知クエリ（Sheets 内で実行）

任意の空きセルに以下を貼ると、同一端末IDからの2回以上の送信が一覧表示されます：

```
=QUERY(online_feedback!A:G, "SELECT E, COUNT(A) WHERE E IS NOT NULL GROUP BY E HAVING COUNT(A) > 1 ORDER BY COUNT(A) DESC", 1)
```

`online_feedback` の部分を `ue_feedback` や `wc_feedback` に変えれば他シートも分析可能。

---

## 9. ファイル構造（このリポジトリ）

```
cotovia-feedback/
├── index.html              # オンライン用（Zoom連携あり）
├── ue/
│   └── index.html          # UE院 店内QR用
├── wc/
│   └── index.html          # WC院 店内QR用
├── .gitignore              # デプロイ対象外（PDF/PNG/master-poster等）
└── DEPLOYMENT.md           # 本ドキュメント

# ローカルのみ（.gitignore で除外）
├── cotovia_feedback_v4.html         # 元ソース（デプロイ前のオリジナル）
├── cotovia_qr_v3.html               # ポスターデザイン（プレースホルダ付き）
├── ue_qr.png, wc_qr.png             # QRコード画像
├── cotovia_qr_{UE,WC}_{A4,A5,A6}.pdf # 印刷用ポスターPDF
└── cotovia_github_pages_setup.md    # 初期セットアップ指示書（過去用）
```

---

## 10. 【保留中】Zoom 3rd-party 廃止後の移行マニュアル

### 10.0 概要
Zoom 3rd-party survey link が廃止されたら、**Zoom内蔵Survey + API同期**にリプレイスする。本格運用時 or 廃止が現実化した時に実施。

### 10.1 Zoom 内蔵 Survey の作成
1. Zoom 左サイドバー → Survey Management → **+ Create Survey**
2. 設定：
   - **Survey Name**: `本日のオンライン診療はいかがでしたか？`
   - **Description**: `本日はご利用ありがとうございました。今後の改善のため、ご感想をお聞かせください。`
   - **質問1** (Rating Scale 1-5): `総合評価をお聞かせください`
   - **質問2** (Long Answer, 500文字): `ご意見・ご感想がございましたらお聞かせください`
3. Save

### 10.2 アカウント設定で Survey をデフォルト指定
1. Account Settings → Meeting → Meeting Survey
2. **Add specified participant survey to all meetings** → ON
3. **Select Survey** → 10.1 で作成した Survey を選ぶ
4. **Allow host to use a 3rd-party survey link** → OFF（混在を避ける）
5. **Who can participate** → External users only
6. Save

### 10.3 Zoom Server-to-Server OAuth アプリ作成
1. https://marketplace.zoom.us/ → Admin ログイン
2. Develop → Build App → **Server-to-Server OAuth**
3. 名前: `Cotovia Feedback Sync`
4. App Credentials タブで以下をメモ：
   - **Account ID**
   - **Client ID**
   - **Client Secret**
5. Scopes タブで以下を追加：
   - `report:read:admin`
   - `meeting:read:admin`
   - `user:read:admin`
6. Activation タブ → Activate

### 10.4 GAS に同期スクリプト追加
GAS プロジェクトで File → New → Script file → 名前 `zoom_sync`
コードは別ファイル `gas_zoom_sync.gs.txt` に保管（このリポジトリのローカル参照）。
主な内容：`syncZoomSurveys()` 関数が `/v2/report/meetings/{uuid}/survey` を叩いて、過去36時間のミーティング全件の Survey 回答を `online_feedback` シートに追記する。

### 10.5 Script Properties に認証情報を登録
GAS → Project Settings → Script Properties：
- `ZOOM_ACCOUNT_ID`: 10.3 でメモした値
- `ZOOM_CLIENT_ID`: 同上
- `ZOOM_CLIENT_SECRET`: 同上
- `ZOOM_HOST_USER_ID`: オンライン診療ホストのZoomアカウント (例: `doctor@cotoviaclinic.com`)

### 10.6 日次トリガー設定
GAS → Triggers → + Add Trigger：
- 関数: `syncZoomSurveys`
- イベント: Time-driven / Day timer / **2am-3am**
- Save

### 10.7 動作確認
- 翌日2時以降、`online_feedback` シートに前日のZoomサーベイ回答が追記されていること
- 手動実行は GASエディタで `syncZoomSurveys` を選んで Run（初回は権限承認）

### 10.8 移行時の注意
- データ反映に**最大1時間のタイムラグ**あり（Zoom内部の集計処理）
- 匿名回答は名前・メアドが空欄になる仕様
- 既存の `online_feedback` シートのスキーマと互換性を維持（端末ID列は空白で書き込み）
- 移行時はGAS側で**重複防止キー**として `zoom:{uuid}|{participant_id}` を使用

### 10.9 廃止前にやっておくと楽な事前準備
- [ ] Zoom Pro プラン継続確認
- [ ] Admin アクセスがあること
- [ ] ホストとなる Zoom ユーザーが固定であること（複数医師の場合は対応拡張必要）

---

## 11. 緊急時の連絡・復旧

GAS が動かない / Sheets が壊れた等で運用が止まったとき：

1. **Sheets のバックアップ**: ファイル → 「コピーを作成」で別ファイルに退避
2. **GAS の旧バージョン復旧**: GAS エディタ → デプロイを管理 → 過去バージョンに戻す
3. **Pages の旧バージョン復旧**: `git log` で過去コミット確認 → `git revert <commit>`

---
