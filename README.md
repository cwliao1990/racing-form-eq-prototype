---
title: Gallop AI PP 元件規格書（Schema 對齊版）
audience: designer + backend
language: zh
status: draft
last-updated: 2026-04-30
schema-source: C:\Users\cwlia\Desktop\gallop_schema_output (2026-04-14 版, 36 tables)
related:
  - pp-feature-gap.md (vs DRF Classic 缺口分析)
  - racing-form-eq.html (prototype)
---

# Gallop AI PP 元件規格書（Schema 對齊版）

寫給網站設計師、前端工程師、後端工程師。逐區塊說明每個欄位的意義、視覺狀態、空值處理，並對齊現有 schema、標出需要新增的欄位。

prototype HTML 檔案：[`index.html`](./index.html)（單檔，無外部依賴）。

---

## 圖例

每個元件的「資料來源」欄位用以下符號標記：

- ✅ **現有 schema 可以直接取**
- 🔁 **現有但需要 join 或計算**
- ➕ **需要 backend 新增**（欄位或聚合邏輯）
- 🤖 **AI / ML 模型產生**（不在現有 schema，由模型輸出）

---

## 0. 整體架構（由上而下）

```
┌─ Navbar
│
├─ Race Header Card .................. 比賽基本資訊（標題、條件、wager）
│
├─ AI Race Preview .................... AI 整場分析（teal 邊條）
│
├─ Race Analysis (Gallop AI) .......... 全場馬匹分析表 + Top 3 預測 + Value Bet
│
├─ Horse Info Card ................... 焦點馬基本資料（重複出現，每匹馬一塊）
│  ├─ Post + 名字 + 顏色配 + Pace + 母父系
│  ├─ Jockey + Trainer 名字與基本統計
│  ├─ Lifetime / Year / Surface 統計表
│  ├─ Jockey Compact Cards（5 張卡片）
│  ├─ Trainer Compact Cards（6 張卡片）
│  └─ AI Horse Summary（purple 邊條）
│
├─ PP Table .......................... 過往賽歷表
│  └─ Prior Trainer Separator Row（換廄時才出現）
│
└─ Works Line ......................... 近期練習紀錄
```

---

## A. 資料來源對齊摘要（Schema Alignment Summary）

對照 schema 的全域差距總覽。下方每個元件的「資料來源」欄會引用這裡的判斷。

### A-1. 已有欄位可直接覆蓋

| 顯示需求 | Schema 對應 |
|---|---|
| 馬場、時區 | `track.name`, `track.timezone` |
| 賽事日、開賽時間 | `meeting.raceDate`, `race.raceTime` |
| 比賽序號、距離、地面、級別、年/性別限制 | `race.seq`, `race.distance`, `race.courseSurface`, `race.type`, `race.grade`, `race.ageRestriction`, `race.sexRestriction` |
| 主獎金、認領金額 | `race.purse`, `race.claimingPrice` |
| 馬名、毛色、性別、生日、父母、breeder | `horse.name`, `horse.color`, `horse.sex`, `horse.foalingDate`, `horse.sire`, `horse.dam`, `horse.breeder` |
| 出閘號、program number、鞍布顏色、賦重 | `runner.postPosition`, `runner.programNumber`, `runner.saddleClothColor`, `runner.weightCarried` |
| 賽前/賽後賠率、裝備、藥物 | `runner.preWinOdds`, `runner.postWinOdds`, `runner.equipment`, `runner.medication` |
| 名次、獎金、賽事評論 | `runner.rank`, `runner.earn`, `runner.comment` |
| 分段時間、分段距離（賽後資料） | `runner.fractionList` (JSON), `runner.pointOfCallList` (JSON) |
| 騎師/訓練師/馬主名字 | `jockey.name`, `trainer.name`, `owner.name`（透過 runner 註冊號碼 join） |
| 生涯統計（出賽/勝/獎金） | `career_race_summary.*` |
| 年度/地面 split 統計 | `race_summary` + `race_summary_by_*` 系列 |

### A-2. 需要 join 或計算的需求

| 顯示需求 | 計算方式 |
|---|---|
| Eligibility 文字 | `race.ageRestriction` + `race.sexRestriction` 合併 |
| Lifetime / Year / Surface 統計表（4×6 表格） | `career_race_summary` + `race_summary_by_year` + `race_summary_by_course_surface` 多筆組合 |
| Sales `Home Bred` 自繁判定 | `horse.breeder == owner.name`（同實體） |
| Trip lengths 落後身位 | `runner.pointOfCallList` JSON 解析 |
| PP rows 賽歷 | `runner_past_performance` (join table) → `pastRunnerId` 鏈到歷史 `runner` 記錄 → `race`, `meeting`, `track` |
| Prior Trainer 歷史換廄 | 比對該馬所有歷史 `runner.trainer` 註冊號碼，偵測換廄點，按 trainer 分組 PP rows |

### A-3. 需要 backend 新增的欄位 / 表 / 邏輯

> ⚠️ 以下都是**現有 schema 沒有**、prototype 有顯示的內容。要決定：(a) 由 data supplier 直接拿 (Equibase？) (b) 自己計算 (c) 暫不實作。

#### A-3a. Race 層級

| 缺項 | 建議 |
|---|---|
| `race.beyerPar`（該級別標準速度指數） | 新增 int 欄位，從 Equibase 取或自行計算 |
| `race.trackRecordTime`（同距離歷史最快） | 新增欄位或建 `track_record_by_distance` 表 |
| `race.purseSupplemental` + `race.supplementalLabel`（如 CBOIF $19,500） | 現有 `race.purse` 是單一 int，附加獎金需新欄位 |
| `race.wagers` JSON（投注種類 + 最低注金 + takeout %） | `race.raceCondition` 是 text，建議解析後存結構化欄位或新表 |
| `race.assignedWeight`（預設賦重） | 目前 runner.weightCarried 是個人值；race-level 預設可能不一定統一。確認業務規則 |

#### A-3b. AI / ML 模型輸出（全部新增）

需要規劃 AI service 並新欄位儲存：

| 欄位 | 說明 |
|---|---|
| `race.aiPreviewText` | 整場 AI 分析文字（HTML 或 Markdown） |
| `race.aiTop3Predictions` JSON | Top 3 預測（含理由） |
| `race.aiValueBet` JSON | Value Bet（含理由列表） |
| `runner.aiSpeedIndex` | Gallop AI 自家速度指數 |
| `runner.aiPaceEarly`, `runner.aiPaceLate` | 早段、晚段速度模型輸出 |
| `runner.aiRunningStyle` | Speed / Stalker / Midpack / Closer / Plodder |
| `runner.aiWinProbability` | 勝率預測 0~1 |
| `runner.aiKeyNote` | 30 字內重點摘要 |
| `runner.aiHorseSummary` | 該馬詳細 AI 分析文字 |
| `runner.cardStatusFlags` JSON | 各情境卡片 hot/warn/normal 判斷結果 |

#### A-3c. PP Row 層級

| 缺項 | 建議 |
|---|---|
| Beyer Speed Figure（每場過往賽事） | 新增 `runner.beyerSpeedFigure`，從 Equibase feed 取 |
| TimeformUS Pace E→L（每場過往賽事） | 新增 `runner.timeformPaceEarly`, `runner.timeformPaceLate` |
| Top finishers 結構化資料 | 目前只有 `runner.comment`，建議新增 `race_summary.topFinishers` JSON |

#### A-3d. 訓練師 / 騎師情境統計（全部新增）

現有 `race_summary_by_trainer` / `race_summary_by_jockey` 只有「按年份」聚合。Prototype 需要的是**按比賽情境**聚合：

**訓練師 6 個情境**（參考 DRF）：

| 情境 | 含義 |
|---|---|
| 61-180D | 馬離場 61~180 天回歸的成績 |
| 1stTurf | 馬首次跑草地的成績 |
| Dirt/Turf | 馬從泥地切換到草地的成績 |
| Turf | 草地賽整體成績 |
| Routes | 中長距離（≥1 mile） |
| MdnSpWt | Maiden Special Weight 級別 |

**騎師 5 個情境**：

| 情境 | 含義 |
|---|---|
| 本年度整體 | 季統計 |
| Turf | 草地賽 |
| 1 Mile | 一英里距離 |
| >6/1 LS | 賠率 >6/1 (longshot) |
| J/T 搭配 | 與當前訓練師組合 |

每組要回傳：`{ starts, wins, winRate, roi2dollar, status: 'normal'|'hot'|'warn' }`

**建議實作**：建立 `trainer_situational_summary`、`jockey_situational_summary` 兩張聚合表，dispatch task 定期更新；或於 query 時動態 SQL 計算（看資料量決定）。

#### A-3e. 其他缺項

| 缺項 | 建議 |
|---|---|
| Sales / 拍賣紀錄 | 新增 `horse.salesRecord` JSON（多筆拍賣）或 `horse_sales` 子表 |
| Equipment change 是否「首次」 | 現有 `runner.equipment` 是當下值，需要與 horse 上一場的 equipment 比對才能判斷「change from previous」 |
| Workouts 練習紀錄 | **目前無 workouts 表**，需新增 `horse_workout` 表（date, track, distance, surface, time, handling, ranking, total） |
| Sire stud fee（馬名後面常見的 $100,000） | 現有 horse.sire 只是字串，需擴充為 sire 物件含 stud fee |
| Silks pattern（鞍布實際圖案） | 目前只有 `runner.saddleClothColor` 文字，要繪製真實 silks 需要結構化 pattern data |
| Foreign race row 格式（歐洲 / 香港等海外賽歷） | 海外賽歷用詞與單位不同（furlongs vs metres、RH vs LH），需要 schema 支援 |

---

## 1. Race Header Card `.race-header-card`

最上方比賽基本資訊區。

### 顯示內容

| 欄位 | 範例 | 資料來源 |
|---|---|---|
| Track | Santa Anita | ✅ `track.name`（透過 `race.meetingId → meeting → track`） |
| Race # | R9 | ✅ `race.seq` |
| Date | Sat May 02 | ✅ `meeting.raceDate` |
| Post Time | 5:15 PT | 🔁 `race.raceTime` (UTC) → 依 `track.timezone` 轉換 |
| Distance | 1 Mile | ✅ `race.distance` |
| Surface | Turf | ✅ `race.courseSurface` |
| Course Type | Inner / Main | ✅ `race.courseType` |
| Class | MAIDEN SPECIAL WEIGHT | 🔁 `race.type` + `race.grade` 合併 |
| Purse 主獎金 | $65,000 | ✅ `race.purse` |
| Purse 附加 | +$19,500 CBOIF | ➕ `race.purseSupplemental`, `race.supplementalLabel` |
| Eligibility | Maidens, Fillies 3YO | 🔁 `race.ageRestriction` + `race.sexRestriction` |
| 預設賦重 | 122 lbs | ➕ `race.assignedWeight` 或從 raceCondition parse |
| Beyer Par | 74 | ➕ `race.beyerPar` |
| Track Record | 1:31³ | ➕ `race.trackRecordTime` 或 track-by-distance 表 |
| Wagers + takeout | Win $2 ... Late Pick 3 $3 (15% takeout) | ➕ `race.wagers` JSON 或解析 `race.raceCondition` |

### 右側徽章

- **賽道圖（SVG）** `.track-diagram`：橢圓賽道示意圖，含起跑點橘點與 FIN 標記。文字標 `1mT · 1 turn`。前端依 `race.distance` + `race.courseSurface` + 馬場彎道資料動態渲染。
  - ➕ 需要 `track.turnCount` 或依距離+馬場推算
- **距離徽章** `.badge-mile`：橘色實心。
- **Race Insured** `.badge-insured`：保險旗標。➕ 需新增 `race.isInsured` boolean

### 視覺重點

- `Beyer Par`、`MAIDEN SPECIAL WEIGHT` 用 `<b>` 粗體
- 整段文字用 `·` 分隔
- 三層 race-sub：核心條件 / wager 設定

---

## 2. AI Race Preview `.ai-section`

AI 自動產生的整場文字分析。

| 欄位 | 資料來源 |
|---|---|
| AI 整場分析文字 | 🤖 ➕ `race.aiPreviewText` |

- **左側 4px teal 邊條** (`#0d9488`) 作為 AI 內容識別
- 標籤 `✨ AI Race Preview`，10px uppercase、teal 色
- 段落文字 12px，行高 1.7

---

## 3. Race Analysis Section `.race-analysis`

整場馬匹綜合分析，分左右兩欄。

### 3a. 左欄：所有馬匹分析表 `.ra-table`

| 欄位 | 範例 | 資料來源 |
|---|---|---|
| # | 5 | ✅ `runner.postPosition` |
| Horse | Soul Sister | 🔁 `runner.registrationNumber` → `horse.name` |
| Speed Index AI | 89 | 🤖 ➕ `runner.aiSpeedIndex` |
| Turf Exp | 5 starts (3 Turf) | 🔁 `race_summary_by_course_surface` 該馬篩選 turf |
| Style | Midpack | 🤖 ➕ `runner.aiRunningStyle` |
| M/L | 5/2 | ✅ `runner.preWinOdds`（賽前賠率，可能就是 morning line） |
| Win% AI | 28.6% | 🤖 ➕ `runner.aiWinProbability` |
| Key Note | 文字 | 🤖 ➕ `runner.aiKeyNote` |

### 3b. 右欄上：Top 3 Prediction `.pred-box`

- 深藍 header
- 三個 row：1st / 2nd / 3rd 圓形徽章（金 / 銀 / 銅色）
- 每 row：馬名 + 馬號 + 短理由

🤖 ➕ `race.aiTop3Predictions` JSON：
```json
[
  { "place": 1, "postPosition": 5, "horseName": "Soul Sister", "reason": "..." },
  ...
]
```

### 3c. 右欄下：Value Bet Box `.ls-box`

- **橘色邊框 2px 強調**，視覺最突出
- Header `⚡ Value Bet`
- 馬名 + 賣點 + 4~5 條 bullet 理由

🤖 ➕ `race.aiValueBet` JSON：
```json
{ "postPosition": 4, "horseName": "Lookin At Diamond", "tagline": "...", "reasons": [...] }
```

---

## 4. Horse Info Card `.horse-card`

焦點馬完整資料區，**每匹馬一塊**，依出閘號排列。

### 4a. 頂部六欄 `.horse-info`（grid-template-columns）

| 欄位區 | 內容 |
|---|---|
| Post | 出閘號（粗體大字） |
| 名字 + 顏色配 + Pace + Aux | 見下方 4b |
| 母父系 | Sire / Dam / Br |
| 騎師 + 訓練師 | J: + 統計、Tr: + 統計 |
| Lifetime / Year 統計 | Life / 2026 / 2025 / SA 四列 |
| Surface 統計 | D.Fst / Wet / Synth / Turf / Dst 五列 |

### 4b. 名字區塊 `.hi-name-block`

```
Ethereal Quality                        ← horse.name
B.f.3(Mar)  L122                        ← horse.color + sex + age + foaling month + runner.weightCarried + Lasix flag
Own: Manzanita Stables LLC              ← runner.owner → owner.name

[15-1] [silk]  Orange, Yellow Inverted  ← runner.preWinOdds + runner.saddleClothColor
       Chevrons, Yellow

Sales: Home Bred (no auction)           ← Aux meta line
| Equip: no change

[Gallop AI] Early 70 · Closer · Late 57 ← runner.aiPaceEarly / aiRunningStyle / aiPaceLate
```

#### Morning Line 賠率徽章 `.hi-ml`

- 深藍底白字 `15-1`
- ✅ `runner.preWinOdds`（**注意**：要確認 preWinOdds 就是 morning line，還是即時賠率）
- 必填欄位

#### Aux Meta Line `.hi-meta-aux`（Sales / Equip）

| 子欄位 | 有資料時 | 自繁無拍賣 | 第三方無拍賣 |
|---|---|---|---|
| Sales | `KEESEP24 $210k`（深紅粗體 `.aux-filled`） | `Home Bred (no auction)`（灰斜體 `.aux-empty`） | `No public sale`（灰斜體） |
| Equip | `Blinkers ON`（深紅粗體 `.aux-filled`） | `no change`（灰斜體 `.aux-empty`） | 同左 |

| 欄位 | 資料來源 |
|---|---|
| Sales | ➕ `horse.salesRecord` JSON 或新增 `horse_sales` 子表 |
| Equip change | 🔁 `runner.equipment`（當前場次）vs 上一場 `runner.equipment` 比對 |
| 自繁判定 | 🔁 `horse.breeder == runner.owner` 名稱比對 |

#### Gallop AI Pace `.gai-pace`

- 淡綠底徽章 + Early/Late 兩個數字
- 中間顯示跑法（如 `Closer`）
- 🤖 ➕ `runner.aiPaceEarly`, `runner.aiPaceLate`, `runner.aiRunningStyle`

### 4c. Pedigree（母父系）

| 欄位 | 資料來源 |
|---|---|
| Sire 名字 | ✅ `horse.sire` |
| Sire 父系 + stud fee | ➕ 需要 sire 實體 join 並補 stud fee 欄位 |
| Dam 名字 + 母父 | 🔁 `horse.dam` + dam 的 sire（需 join horse 自身） |
| Breeder + 出生州 | ✅ `horse.breeder`（出生州可能要 parse） |

### 4d. Lifetime / Year / Surface 統計表

兩個小 grid 各一欄。

| 欄位 | 資料來源 |
|---|---|
| Life 出賽/勝/位/秀/獎金 | ✅ `career_race_summary` 該馬一筆 |
| 2026 / 2025 等年度 | ✅ `race_summary_by_year` 該馬多筆 |
| 該場馬場（如 SA） | ✅ `race_summary_by_track` 該馬篩選 SA |
| D.Fst / Wet / Synth / Turf | ✅ `race_summary_by_course_surface` 該馬多筆 |
| Dst 同距離成績 | ✅ `race_summary_by_distance` 該馬篩選同距離 |
| 最佳速度指數（每列右側） | ➕ 各 summary 表需新增 `bestBeyer` 欄位 |

---

## 5. Jockey Compact Cards `.jky-compact`

騎師進階統計，**5 張卡片**橫排。

### 結構

```
[Gallop AI] Frey K  |  [card] [card] [card] [card] [card]
```

| 卡片 | label | 含義 | 資料來源 |
|---|---|---|---|
| 1 | 2026 | 騎師本年整體 | ✅ `race_summary_by_year`（依 jockey 篩） |
| 2 | Turf | 在草地賽的成績 | 🔁 `race_summary_by_course_surface` + jockey 篩選（建議建 `jockey_situational_summary` 聚合表） |
| 3 | 1 Mile | 在一英里距離的成績 | 🔁 同上，by distance |
| 4 | >6/1 LS | 賠率 >6/1 (longshot) 的成績 | ➕ 現有 schema 無此聚合，需 SQL 動態算或新增聚合表 |
| 5 | J/T Drysdale | 與當前訓練師搭配的成績 | ➕ 同上，需新增 jockey-trainer pair summary |

### 卡片三層

- **jc-lbl**（最上）：情境名，8.5px 灰色 uppercase
- **jc-sts**（中）：出賽 `162 sts`，9.5px
- **jc-win**（下）：`13W · 8%`，12px 粗體 teal

### 視覺狀態

| Class | 背景 | 文字色 | 用途 |
|---|---|---|---|
| 預設 | 淡綠 `#f0fdf9` | teal | 一般 |
| `.jky-hot` | 淺橘 `#fff8f0` | 橘 `#f47c20` | 強項 + ★ 星號 |
| `.jky-warn` | 淡紅 `#fef2f2` | 紅 `#c00` | 弱項 |

🤖 ➕ 每張卡片回傳 `{ starts, wins, winRate, status: 'normal'|'hot'|'warn' }`，status 由 backend 算（例如勝率 > 季平均 1.5x = hot）。

---

## 6. Trainer Compact Cards `.trn-compact`

訓練師進階統計，**6 張卡片**橫排。視覺與騎師區塊一致。

| 卡片 | label | 含義 | 資料來源 |
|---|---|---|---|
| 1 | 61-180D | 馬離場 61~180 天回歸 | ➕ 需 `trainer_situational_summary` 表 |
| 2 | 1st Turf | 馬首次跑草地 | ➕ 同上 |
| 3 | Dirt/Turf | 泥地切換到草地 | ➕ 同上 |
| 4 | Turf | 草地賽整體 | 🔁 `race_summary_by_course_surface` + trainer 篩選 |
| 5 | Routes | 中長距離（≥1 mile） | 🔁 `race_summary_by_distance` + trainer 篩選 |
| 6 | MdnSpWt | Maiden Special Weight 級別 | ➕ 需聚合表（race.type 篩選） |

### 卡片三層

- **tc-lbl**（最上）：情境名
- **tc-sts**（中）：出賽 `27 sts`
- **tc-win**（下）：`15% · $2.27`（勝率 + $2 投資 ROI）

### ROI 判讀

`tc-win` 第二個值是 **$2 ROI**：每押 $2 win 平均回收金額。

| 值 | 含義 | 視覺建議 |
|---|---|---|
| > $2.00 | 正期望值 | hot 候選 |
| = $2.00 | 損益兩平 | normal |
| < $2.00 | 負期望值 | warn 候選 |

➕ ROI 計算需要每場 odds + 結果，現有 `runner.preWinOdds` + `runner.rank` 可推，但要建聚合邏輯。

### 視覺狀態（與 jockey 完全一致）

`.trn-hot` / `.trn-warn` 同 jockey 規則。

---

## 7. AI Horse Summary `.ai-section.ai-section-horse`

AI 對單一馬的文字總結。

| 欄位 | 資料來源 |
|---|---|
| AI 個馬分析文字 | 🤖 ➕ `runner.aiHorseSummary` |

- **左側 4px purple 邊條**（`#7c3aed`）區別於整場 AI Preview
- 標籤 `💡 AI Horse Summary` purple 色

---

## 8. PP Table `.pp-table`

過往賽歷，每場一 row。21 欄。

### 取資料邏輯

```
runner_past_performance
  ├── runnerId（當前出賽）
  └── pastRunnerId（歷史 runner 記錄）
       ├── runner（歷史出賽詳情：賠率、名次、分段時間、評論）
       ├── race（歷史比賽：距離、地面、級別、獎金、時間）
       ├── meeting → track（馬場代碼、時區）
       └── jockey, trainer（透過註冊號碼 join）
```

每筆 PP row 需要 join 5 張表才能渲染。

### 完整欄位對照

| # | Class | Header | 範例 | 資料來源 |
|---|---|---|---|---|
| 1 | `pp-date` | Date | 8Jan26 | ✅ `meeting.raceDate`（pastRunner 鏈） |
| 2 | `pp-trk` | Rc/Trk | 4SA | 🔁 `race.seq` + `track.code`（pastRunner 鏈） |
| 3 | `pp-dist` | Dist | 6f / 1 1/16 | ✅ `race.distance` |
| 4 | `pp-surf` | Sf | fst / fm | ✅ `race.courseSurface`（值需要 normalize 成代碼） |
| 5 | `pp-class` | Class | Md Sp Wt 72k | 🔁 `race.type` + `race.grade` + `race.purse / 1000` |
| 6 | `pp-cond` | Cond | C / - | ✅ `race.trackCondition` |
| 7 | `pp-post` | PP | 4 | ✅ `runner.postPosition` |
| 8 | `pp-q1` | 1/4 | :22³ | 🔁 `runner.fractionList` JSON 第 1 段 |
| 9 | `pp-q2` | 1/2 | :45⁴ | 🔁 `runner.fractionList` JSON 第 2 段 |
| 10 | `pp-q3` | 3/4 | :58 / 1:11 | 🔁 `runner.fractionList` JSON 第 3 段 |
| 11 | `pp-str` | Str | 7¹¾ | 🔁 `runner.pointOfCallList` JSON 第 4 點（位置 + lengths） |
| 12 | `pp-fin` | Fin | 4²¾ | 🔁 `runner.pointOfCallList` JSON 最末點 |
| 13 | `pp-jkey` | Jockey | Berrios H | 🔁 `runner.jockey` → `jockey.name` |
| 14 | `pp-wt` | Wt | L122f | 🔁 `runner.weightCarried` + `runner.medication` (L) + `runner.equipment` (b/f) |
| 15 | `pp-odds` | Odds | 48.60 | ✅ `runner.postWinOdds`（or preWinOdds 若 post 無） |
| 16 | `pp-rnrs` | #Rnrs | 10 | ✅ `race.numberOfRunners` |
| 17 | `pp-time` | Time | 1:11² | 🔁 `runner.fractionList` 最末段 |
| 18 | `pp-beyer` | Speed Index | 58 | ➕ `runner.beyerSpeedFigure`（需新增） |
| 19 | `pp-pace` | Pace E→L | 81→15 | ➕ `runner.timeformPaceEarly`, `runner.timeformPaceLate`（需新增） |
| 20 | `pp-purse` | Purse | $72k | ✅ `race.purse / 1000` |
| 21 | `pp-comment` | Top Finishers | 文字 | 🔁 `runner.comment`（trip comment） + 需新增 top finishers 結構化資料 |

### 細節：Wt 欄藥物/裝備標記

格式 `[L][weight][med_tags]`：

| 字元 | 含義 | 來源 |
|---|---|---|
| `L` (前綴) | Lasix 速尿劑 | `runner.medication` 含 'L' |
| `b` (後綴) | bandages back 後綁帶 | `runner.equipment` 含 'b' |
| `f` (後綴) | bandages front 前綁帶 | `runner.equipment` 含 'f' |

⚠️ 現有 `runner.medication` 與 `runner.equipment` 是 varchar，需要確認儲存格式（CSV / 代碼 / 文字）。

### 細節：Str / Fin 欄落後身位上標

從 `runner.pointOfCallList` JSON 解析，每個 call 點有 `position` + `lengthsBack`：

```json
[
  { "call": "start", "position": 8, "lengthsBack": null },
  { "call": "1/4",   "position": 8, "lengthsBack": 2.75 },
  { "call": "1/2",   "position": 8, "lengthsBack": 2.75 },
  { "call": "str",   "position": 7, "lengthsBack": 1.75 },
  { "call": "fin",   "position": 4, "lengthsBack": 2.75 }
]
```

➕ 需確認現有 JSON 是否含 lengthsBack 欄位。如果沒，需要 backend parse Equibase trip line 補上。

### 細節：Pace E→L 欄

`81→15` = 早段速度 81、晚段速度 15。箭頭只是視覺，數字大小由閱讀者判讀：

- 早段高、晚段低 = 起跑快但後段沒勁
- 早段低、晚段高 = 後段衝得起來
- 速度未達標 / eased = 顯示 `–`（en-dash）

➕ TimeformUS Pace 是第三方資料，需要授權或自行計算。

---

## 9. Prior Trainer Separator Row `.pp-prior-row`

PP 表中的**跨欄分隔列**，仿 DRF 排版。

### 何時渲染

僅在馬曾經換過訓練師時出現。出現位置：「前任訓練師期間 PP rows」**正上方**。

### 視覺

- 淡黃底 `#fffbe6`、上下虛線邊（黃色）
- 黃色徽章 `PREV TRAINER`
- 文字格式：`Previously trained by [名字] [年]: ([starts] sts · [W-P-S] · Win [%])`

### 資料來源

| 欄位 | 來源 |
|---|---|
| 換廄偵測邏輯 | 🔁 該馬所有歷史 `runner.trainer` 比對，找連續變更點 |
| 前訓練師整年統計 | 🔁 `race_summary_by_year` + `race_summary_by_trainer` 該訓練師當年資料 |
| 應用到哪幾筆 PP rows | ➕ 需 backend 計算 `applies_to_pp_indices` 陣列 |

### 多次換廄

若有多任前訓練師，會出現多條分隔列，由近到遠依序。**無換廄則完全不渲染這列**。

> prototype HTML 中 #1 Ethereal Quality 無換廄紀錄，但目前在 PP rows 中間插了一條 `[demo row]` 標註的示範列，**正式版該馬無換廄就不渲染**。

---

## 10. Works Line `.works-line`

最近練習紀錄，單行文字串。

格式：`[日期] [馬場] [距離] [場地] [時間][符號] [排名]/[總馬數]`

範例：`27Apr26 SA 4f fst :49H 10/18`

| 字元 | 含義 |
|---|---|
| `H` 後綴 | handily（鬆勒慢練） |
| `B` 後綴 | breezing（中等強度） |
| `Hg` | handily out of gate（出閘練） |
| `10/18` | 該日同距離操作中排名第 10、共 18 匹 |

| 欄位 | 資料來源 |
|---|---|
| Workouts 全部 | ➕ **目前無 workouts 表**，需新增 `horse_workout` 表 |

建議 `horse_workout` schema：

```sql
CREATE TABLE horse_workout (
  id INT PRIMARY KEY AUTO_INCREMENT,
  horseId INT NOT NULL,
  workDate DATE NOT NULL,
  trackCode VARCHAR(255) NOT NULL,
  distance VARCHAR(255) NOT NULL,
  surface VARCHAR(255) NOT NULL,
  time VARCHAR(255) NOT NULL,    -- ":49"
  handlingCode VARCHAR(10),       -- "H", "B", "Hg"
  ranking INT,
  totalWorks INT,
  -- system fields
  createTime TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ...
  KEY (horseId, workDate)
);
```

---

## 11. 全域視覺語意

### 色彩系統

| 顏色 | 用途 | 範例位置 |
|---|---|---|
| 深藍 `#0d1f35` | 主題色 / 標題 / TRAINER 徽章 / ML 賠率徽章 | navbar, race title, ml-badge |
| 橘 `#f47c20` | 強調 / CTA / 注意力導引 | logo, value bet box, hot 卡片 |
| Teal `#0d9488` | AI 內容識別 / 卡片基底 | AI Preview 邊條, jky-badge, 卡片背景 |
| Purple `#7c3aed` | AI Horse Summary 識別 | horse summary 邊條 |
| 紅 `#c00` | 重要數字 / warn 狀態 | Beyer figure, warn 卡片 |
| 黃 `#d4b955` | Prior Trainer 分隔列 | pp-prior-row |
| 灰系 | 一般文字、disabled、empty 狀態 | aux-empty, ra-dim |

### 三態卡片狀態（jky / trn 共用）

```
.normal  ──→ 淡綠底，teal 字
.hot     ──→ 淡橘底，橘字 + ★ 星號
.warn    ──→ 淡紅底，紅字
```

判斷邏輯由 backend 決定 status 字串，前端只負責 class 對應。

---

## 12. CSS Class 速查（給前端）

```
.race-header-card          比賽 header
.ai-section                AI 區塊（+ .ai-section-horse 變體）
.race-analysis             整場分析
  .ra-table                所有馬綜合表
  .pred-box                Top 3 Prediction
  .ls-box                  Value Bet
.horse-card                焦點馬區塊
  .horse-info              頂部 6 欄 grid
  .hi-name-block           馬名與配色區
  .hi-ml                   ML 賠率徽章
  .hi-meta-aux             Sales / Equip aux 行
  .gai-pace                Gallop AI Pace 行
.jky-compact               騎師卡片區
  .jky-card                騎師單卡（+ .jky-hot / .jky-warn 變體）
.trn-compact               訓練師卡片區
  .trn-card                訓練師單卡（+ .trn-hot / .trn-warn 變體）
.pp-table                  PP 表
  .pp-prior-row            換廄分隔列
.works-line                Works 行
```

---

## 13. 給後端工程師的整合摘要

要把 prototype 跑起來，需要的工作分三類。

### 13a. 純查詢（schema 已支援）

可以直接用現有表 + join 寫 query：

- Race header 大部分欄位（除了 Beyer Par、track record、wagers、supplemental purse）
- Horse 基本資料、pedigree
- Lifetime / Year / Surface 統計（用 `career_race_summary` + `race_summary_*` 系列）
- PP rows 大部分欄位（除了 Beyer figure、TimeformUS pace）
- Comment / trip comment

### 13b. 需要新增的 schema 變更

| 優先級 | 項目 |
|---|---|
| **P0** | `runner.beyerSpeedFigure`、`race.beyerPar`（核心 handicapping 數字） |
| **P0** | AI 模型輸出欄位（`runner.aiSpeedIndex`, `aiPaceEarly/Late`, `aiRunningStyle`, `aiWinProbability`, `aiKeyNote`, `aiHorseSummary`, `race.aiPreviewText`, `race.aiTop3Predictions`, `race.aiValueBet`） |
| **P1** | `trainer_situational_summary`、`jockey_situational_summary` 兩張聚合表（6+5 個情境） |
| **P1** | `horse_workout` 表 |
| **P2** | `runner.timeformPaceEarly/Late`、`race.trackRecordTime`、`race.purseSupplemental` |
| **P2** | `horse.salesRecord` 或 `horse_sales` 子表 |
| **P3** | Sire stud fee 擴充、結構化 silks pattern、wagers 結構化 |

### 13c. 需要演算法 / 業務邏輯

| 邏輯 | 說明 |
|---|---|
| 換廄偵測 | 比對該馬所有歷史 `runner.trainer`，找連續變更點，分組 PP rows |
| 自繁判定 | `horse.breeder == owner.name`（Home Bred 文案） |
| Equipment change 判定 | 與該馬上一場 `runner.equipment` 比對 |
| Pace fraction lengths 解析 | `runner.pointOfCallList` JSON 取 lengthsBack |
| Card status (hot/warn) 判定 | 各情境統計 vs 季平均、ROI vs $2 損益點 |

### 13d. 樣本 API response 結構（理想版）

```jsonc
GET /api/race/:raceId
{
  "race": {
    "id": 12345,
    "trackName": "Santa Anita",
    "trackCode": "SA",
    "timezone": "America/Los_Angeles",
    "raceDate": "2026-05-02",
    "raceTime": "2026-05-02T17:15:00-07:00",
    "seq": 9,
    "distance": "1 Mile",
    "courseSurface": "Turf",
    "courseType": "Inner",
    "type": "MAIDEN SPECIAL WEIGHT",
    "grade": null,
    "purse": 65000,
    "purseSupplemental": 19500,         // ➕ 新增
    "supplementalLabel": "CBOIF",       // ➕ 新增
    "ageRestriction": "3 Year Olds",
    "sexRestriction": "Fillies",
    "assignedWeight": 122,              // ➕ 新增
    "beyerPar": 74,                     // ➕ 新增
    "trackRecordTime": "1:31.6",        // ➕ 新增
    "wagers": [...],                    // ➕ 新增（結構化）
    "aiPreviewText": "...",             // 🤖 ➕
    "aiTop3Predictions": [...],         // 🤖 ➕
    "aiValueBet": {...}                 // 🤖 ➕
  },
  "entries": [
    {
      "runnerId": 67890,
      "postPosition": 1,
      "horse": {
        "name": "Ethereal Quality",
        "color": "B",
        "sex": "f",
        "foalingDate": "2023-03-15",
        "sire": {...},                  // 🔁 join + ➕ stud fee
        "dam": {...},                   // 🔁 join
        "breeder": "Manzanita Stables LLC",
        "salesRecord": null             // ➕ 新增
      },
      "owner": "Manzanita Stables LLC",
      "isHomeBred": true,                // 🔁 計算
      "weightCarried": 122,
      "lasix": true,
      "saddleClothColor": "Orange...",
      "preWinOdds": "15-1",
      "equipment": "f",
      "equipmentChangeFromPrevious": null,  // 🔁 計算
      "jockey": { "name": "Frey K", ... },
      "trainer": { "name": "Drysdale Neil", ... },
      "aiSpeedIndex": 81,                // 🤖 ➕
      "aiPaceEarly": 70,                 // 🤖 ➕
      "aiPaceLate": 57,                  // 🤖 ➕
      "aiRunningStyle": "Closer",        // 🤖 ➕
      "aiWinProbability": 0.063,         // 🤖 ➕
      "aiKeyNote": "...",                // 🤖 ➕
      "aiHorseSummary": "...",           // 🤖 ➕
      "lifetimeStats": [...],            // 🔁 from career_race_summary + summary tables
      "surfaceStats": [...],             // 🔁
      "jockeyStats": {                   // ➕ jockey_situational_summary
        "season": {...},
        "turf": {...},
        "distance1Mile": {...},
        "longshot": {...},
        "withTrainer": {...}
      },
      "trainerStats": {                  // ➕ trainer_situational_summary
        "layoff61to180": {...},
        "firstTurf": {...},
        "dirtToTurf": {...},
        "turf": {...},
        "routes": {...},
        "mdnSpWt": {...}
      },
      "priorTrainers": [],               // 🔁 計算
      "ppRows": [
        {
          "raceDate": "2026-01-08",
          "trackCode": "SA",
          "raceSeq": 4,
          "distance": "6f",
          "courseSurface": "fst",
          "raceType": "Md Sp Wt 72k",
          "trackCondition": "C",
          "postPosition": 4,
          "fractionList": [...],         // ✅ JSON
          "pointOfCallList": [...],      // ✅ JSON（含 lengthsBack）
          "jockey": "Berrios H",
          "weightCarried": 122,
          "medication": "L",
          "equipment": "f",
          "closingOdds": "48.60",
          "fieldSize": 10,
          "finalTime": "1:11.2",
          "beyerSpeedFigure": 58,        // ➕ 新增
          "timeformPaceEarly": 81,       // ➕ 新增
          "timeformPaceLate": 15,        // ➕ 新增
          "purse": 72000,
          "topFinishers": [...],         // ➕ 結構化
          "tripComment": "Off slow, 2-5w, kept on"
        }
      ],
      "works": [...]                     // ➕ from horse_workout
    }
  ]
}
```

---

## 14. Open Questions（需業務 / 後端確認）

1. **`runner.preWinOdds` 是 morning line 還是賽前最後賠率？** 名字看起來是後者。如果是後者，就需要另外存 morning line。
2. **`runner.fractionList` / `runner.pointOfCallList` JSON 內 schema** 確切是什麼？是否含 lengthsBack？
3. **TimeformUS / Beyer 是否已有授權**？兩者都是第三方資料，影響 P0/P2 排程。
4. **AI 模型** 目前在哪個階段？是已可呼叫的 service，還是要新建？
5. **資料供應商** 是 Equibase / DRF / 自有 scraper？影響 Sales、Workouts、Beyer 取得方式。
6. **海外賽歷**（歐洲、香港、日本）佔比？是否 MVP 必須支援？
