---
title: Gallop AI PP 元件規格書
audience: designer + backend
language: zh
status: draft
last-updated: 2026-04-30
related:
  - pp-feature-gap.md (vs DRF Classic 缺口分析)
  - racing-form-eq.html (prototype)
---

# Gallop AI PP 元件規格書

寫給網站設計師、前端工程師、後端工程師。逐區塊說明每個欄位的意義、視覺狀態、空值處理、與後端資料對應建議。

prototype HTML 檔案：`racing-form-eq.html`（單檔，無外部依賴）。

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

## 1. Race Header Card `.race-header-card`

最上方比賽基本資訊區。

### 顯示內容

| 欄位 | 範例 | 後端欄位建議 | 說明 |
|---|---|---|---|
| Track | Santa Anita | `race.track_name` | 馬場全名 |
| Race # | R9 | `race.race_number` | 第幾場 |
| Date | Sat May 02 | `race.date` | 比賽日期 |
| Post Time | 5:15 PT | `race.post_time` | 開閘時間（24 小時制 + 時區） |
| Distance | 1 Mile | `race.distance` | 距離 |
| Surface | Turf | `race.surface` | 場地：Dirt / Turf / Synth |
| Class | MAIDEN SPECIAL WEIGHT | `race.class_name` | 賽事級別（粗體強調） |
| Purse | $65,000 (+$19,500 CBOIF) | `race.purse_main`, `race.purse_supplemental` | 總獎金（主獎金+附加獎金） |
| Eligibility | Maidens, Fillies 3YO | `race.eligibility_text` | 參賽條件 |
| Weight | 122 lbs | `race.assigned_weight` | 預設賦重 |
| Beyer Par | 74 | `race.beyer_par` | 此級別歷史標準速度指數（粗體強調） |
| Track Record | 1:31³ | `race.track_record_time` | 同距離歷史最快 |
| Wagers | Win $2, Place $2... Late Pick 3 $3 (15% takeout) | `race.wager_options[]` | 投注種類最低注金 + takeout |

### 右側徽章

- **賽道圖（SVG）** `.track-diagram`：橢圓賽道示意圖，含起跑點橘點與 FIN 標記。文字標 `1mT · 1 turn`（1 mile turf, 1 turn）。後端只需提供 `race.distance` + `race.surface` + `race.turns`，前端動態渲染。
- **距離徽章** `.badge-mile`：橘色實心，醒目顯示距離。
- **Race Insured** `.badge-insured`：保險旗標（如有開放）。

### 視覺重點

- `Beyer Par`、`MAIDEN SPECIAL WEIGHT` 用 `<b>` 粗體
- 整段文字用 `·` 分隔，避免過多換行
- 三層 race-sub 分別放：核心條件 / wager 設定

---

## 2. AI Race Preview `.ai-section`

AI 自動產生的整場文字分析。

- **左側 4px teal 邊條**（`#0d9488`）作為 AI 內容識別
- 標籤 `✨ AI Race Preview`，10px uppercase、teal 色
- 段落文字 12px，行高 1.7
- 兩個段落以上時用 `<p>` 切

**後端**：`race.ai_preview_text`（HTML 或 Markdown）。

---

## 3. Race Analysis Section `.race-analysis`

整場馬匹綜合分析，分左右兩欄。

### 3a. 左欄：所有馬匹分析表 `.ra-table`

| 欄位 | 範例 | 後端欄位 | 說明 |
|---|---|---|---|
| # | 5 | `entry.post_position` | 出閘號 |
| Horse | Soul Sister | `entry.horse_name` | 馬名 |
| Speed Index AI | 89 | `entry.gallop_speed_index` | AI 計算的速度指數（紅色強調），標 `AI` 小角標 |
| Turf Exp | 5 starts (3 Turf) | `entry.turf_experience_text` | 草地經驗摘要 |
| Style | Midpack | `entry.running_style` | 跑法：Speed / Stalker / Midpack / Closer / Plodder |
| M/L | 5/2 | `entry.morning_line` | 晨盤賠率，前 3 contender 用粗體深藍 |
| Win% AI | 28.6% | `entry.gallop_win_probability` | AI 預測勝率（綠色強調），前 3 用深綠粗體 |
| Key Note | 文字 | `entry.key_note` | 30 字內重點摘要 |

### 3b. 右欄上：Top 3 Prediction `.pred-box`

- 深藍 header
- 三個 row，分別 1st / 2nd / 3rd 圓形徽章（金 / 銀 / 銅色）
- 每 row：馬名 + 馬號 + 短理由

**後端**：`race.gallop_top3[]`（排序好的 3 個物件）。

### 3c. 右欄下：Value Bet Box `.ls-box`

- **橘色邊框 2px 強調**，視覺最突出（CTA 等級）
- Header `⚡ Value Bet`
- 馬名 + 賣點（如 `Speed Index 90 (Field Best)`）
- 4~5 條 bullet 列舉理由

**後端**：`race.gallop_value_bet`（單一物件）。

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
Ethereal Quality                        ← 馬名 16px 粗體
B.f.3(Mar)  L122                        ← 毛色性別年齡(出生月) + 賦重
Own: Manzanita Stables LLC              ← 馬主

[15-1] [silk]  Orange, Yellow Inverted  ← ML 賠率徽章 + 顏色配
       Chevrons, Yellow

Sales: Home Bred (no auction)           ← Aux meta line
| Equip: no change

[Gallop AI] Early 70 · Closer · Late 57 ← Gallop AI Pace 行
```

#### Morning Line 賠率徽章 `.hi-ml`

- 深藍底白字 `15-1`
- 後端：`entry.morning_line`，格式 `{n}-{m}` 或 `{n}/{m}`
- 必填欄位

#### Aux Meta Line `.hi-meta-aux`（Sales / Equip）

| 子欄位 | 有資料時 | 自繁無拍賣 | 第三方無拍賣 |
|---|---|---|---|
| Sales | `KEESEP24 $210k`（深紅粗體 `.aux-filled`） | `Home Bred (no auction)`（灰斜體 `.aux-empty`） | `No public sale`（灰斜體） |
| Equip | `Blinkers ON`（深紅粗體 `.aux-filled`） | `no change`（灰斜體 `.aux-empty`） | 同左 |

**後端欄位**：
- `entry.sales_record`（可空字串）
- `entry.equipment_change`（可空字串）
- `entry.is_home_bred`（boolean，用來決定空值文字）

#### Gallop AI Pace `.gai-pace`

- 淡綠底徽章 + Early/Late 兩個數字
- 中間顯示跑法（如 `Closer`）
- 後端：`entry.gallop_pace_early`, `entry.gallop_pace_late`, `entry.running_style`

### 4c. 統計表（Lifetime + Surface）

兩個表格各一欄，用 `.sr` flex row 排版。每 row 6 欄：

```
[Label]  [出賽]  [M]  [W-P-S 三個數字]  [獎金]  [最佳速度指數]
Life     3       M    0  1              $12,800 58
```

- M = Maiden 字母標記（未破冠）
- 最佳速度指數紅字粗體
- 後端：標準 race history aggregate

---

## 5. Jockey Compact Cards `.jky-compact`

騎師進階統計，**5 張卡片**橫排。

### 結構

```
[Gallop AI] Frey K  |  [card] [card] [card] [card] [card]
```

| 卡片 | label | 含義 | 後端欄位 |
|---|---|---|---|
| 1 | 2026 | 騎師本年整體（出賽/勝/勝率） | `jockey.season_starts`, `season_wins`, `season_win_rate` |
| 2 | Turf | 在草地賽的成績 | `jockey.surface_turf_*` |
| 3 | 1 Mile | 在一英里距離的成績 | `jockey.distance_1mile_*` |
| 4 | >6/1 LS | 賠率 >6/1 (longshot) 的成績 | `jockey.longshot_*` |
| 5 | J/T Drysdale | 與當前訓練師搭配的成績 | `jockey.with_trainer_*` |

### 卡片三層

- **tc-lbl**（最上）：情境名，8.5px 灰色 uppercase
- **tc-sts**（中）：出賽 `162 sts`，9.5px
- **tc-win**（下）：`13W · 8%`，12px 粗體 teal

### 視覺狀態（重要）

| Class | 背景 | 文字色 | 用途 |
|---|---|---|---|
| 預設 | 淡綠 `#f0fdf9` | teal | 一般 |
| `.jky-hot` | 淺橘 `#fff8f0` | 橘 `#f47c20` | 強項（如 J/T 17%）+ ★ 星號 |
| `.jky-warn` | 淡紅 `#fef2f2` | 紅 `#c00` | 弱項（如 >6/1 LS 6%） |

**後端建議**：每張卡片回 `{ stats, status: 'normal' | 'hot' | 'warn' }`。判斷規則由後端決定（例如 ROI > $2 或勝率高於季平均 1.5x = hot；勝率低於季平均 0.7x = warn）。

---

## 6. Trainer Compact Cards `.trn-compact`

訓練師進階統計，**6 張卡片**橫排。視覺與騎師區塊一致。

### 結構

```
[TRAINER] Drysdale Neil  |  [card] [card] [card] [card] [card] [card]
```

| 卡片 | label | 含義 | 後端欄位 |
|---|---|---|---|
| 1 | 61-180D | 訓練師讓馬離場 61~180 天回歸的成績 | `trainer.layoff_61_180_*` |
| 2 | 1st Turf | 馬首次跑草地的成績 | `trainer.first_turf_*` |
| 3 | Dirt/Turf | 馬從泥地切換到草地 | `trainer.dirt_to_turf_*` |
| 4 | Turf | 草地賽整體 | `trainer.surface_turf_*` |
| 5 | Routes | 中長距離（≥1 mile） | `trainer.routes_*` |
| 6 | MdnSpWt | Maiden Special Weight 級別 | `trainer.mdnspwt_*` |

### 卡片三層（與 jockey 略不同）

- **tc-lbl**（最上）：情境名
- **tc-sts**（中）：出賽 `27 sts`
- **tc-win**（下）：`15% · $2.27`（勝率 + $2 投資 ROI）

### ROI 判讀（給後端）

`tc-win` 第二個值是 **$2 ROI**：每押 $2 win 平均回收金額。

| 值 | 含義 | 視覺建議 |
|---|---|---|
| > $2.00 | 正期望值（賺） | hot 候選 |
| = $2.00 | 損益兩平 | normal |
| < $2.00 | 負期望值（虧） | warn 候選 |

### 視覺狀態（與 jockey 完全一致）

| Class | 背景 | 文字色 | 用途 |
|---|---|---|---|
| 預設 | 淡綠 `#f0fdf9` | teal | 一般 |
| `.trn-hot` | 淺橘 `#fff8f0` | 橘 | ROI > $2 且勝率高 |
| `.trn-warn` | 淡紅 `#fef2f2` | 紅 | ROI < $1 或勝率明顯低 |

---

## 7. AI Horse Summary `.ai-section.ai-section-horse`

AI 對單一馬的文字總結。

- **左側 4px purple 邊條**（`#7c3aed`）區別於整場 AI Preview 的 teal
- 標籤 `💡 AI Horse Summary` purple 色
- 後端：`entry.ai_summary_text`

---

## 8. PP Table `.pp-table`

過往賽歷，每場一 row。21 欄。

### 完整欄位對照

| # | Class | Header | 範例 | 含義 | 後端欄位 |
|---|---|---|---|---|---|
| 1 | `pp-date` | Date | 8Jan26 | 比賽日期 `DDMMMYY` | `pp.race_date` |
| 2 | `pp-trk` | Rc/Trk | 4SA | **Race # + 馬場縮寫**（4SA = 第 4 場 SA） | `pp.race_number`, `pp.track_code` |
| 3 | `pp-dist` | Dist | 6f / 1 1/16 | 距離 | `pp.distance` |
| 4 | `pp-surf` | Sf | fst / fm | 場地代碼：fst=Fast Dirt, fm=Firm Turf, sy=Sloppy 等 | `pp.surface_code` |
| 5 | `pp-class` | Class | Md Sp Wt 72k | 級別 + 獎金 | `pp.class_name`, `pp.purse_k` |
| 6 | `pp-cond` | Cond | C / - | 場地特殊條件（C = Came back to dirt 等） | `pp.condition_code` |
| 7 | `pp-post` | PP | 4 | 出閘號 | `pp.post_position` |
| 8 | `pp-q1` | 1/4 | :22³ | 1/4 mile 分段時間 | `pp.split_quarter` |
| 9 | `pp-q2` | 1/2 | :45⁴ | 1/2 mile 分段時間 | `pp.split_half` |
| 10 | `pp-q3` | 3/4 | :58 / 1:11 | 3/4 mile 分段時間 | `pp.split_three_quarter` |
| 11 | `pp-str` | Str | 7¹¾ | **Stretch 名次 + 落後身位上標** | `pp.stretch_position`, `pp.stretch_lengths_back` |
| 12 | `pp-fin` | Fin | 4²¾ | **終點名次 + 落後身位上標** | `pp.finish_position`, `pp.finish_lengths_back` |
| 13 | `pp-jkey` | Jockey | Berrios H | 騎師 | `pp.jockey_name` |
| 14 | `pp-wt` | Wt | L122f | **賦重 + 藥物/裝備標記** | 見下表 |
| 15 | `pp-odds` | Odds | 48.60 | 收盤賠率 | `pp.closing_odds` |
| 16 | `pp-rnrs` | #Rnrs | 10 | 出賽馬數 | `pp.field_size` |
| 17 | `pp-time` | Time | 1:11² | 完賽時間 | `pp.final_time` |
| 18 | `pp-beyer` | Speed Index | 58 | Beyer Speed Figure，紅字粗體 | `pp.beyer_speed_figure` |
| 19 | `pp-pace` | Pace E→L | 81→15 | **TimeformUS Pace**：早段速度→晚段速度 | `pp.timeform_pace_early`, `pp.timeform_pace_late` |
| 20 | `pp-purse` | Purse | $72k | 該場獎金 | `pp.purse_k` |
| 21 | `pp-comment` | Top Finishers | 文字 | 前 3 名 + trip comment | `pp.top_finishers_text`, `pp.trip_comment` |

### 細節：Wt 欄藥物/裝備標記

格式 `[L][weight][med_tags]`：

| 字元 | 含義 |
|---|---|
| `L` (前綴) | Lasix（速尿劑）使用 |
| `b` (後綴) | bandages back 後綁帶 |
| `f` (後綴) | bandages front 前綁帶 |

範例：`L122f` = 用 Lasix、賦重 122 磅、戴前綁帶。

### 細節：Str / Fin 欄落後身位上標

`<sup class="pp-len">` 用淺灰小字標出。例：

- `4²¾` = 第 4 名，落後 2¾ 身
- `7¹` = 第 7 名，落後 1 身
- `5⁵` = 第 5 名，落後 5 身（差距很大）
- `eased` = 騎手讓馬鬆下來不再追（無 lengths）

### 細節：Pace E→L 欄

`81→15` = 早段速度 81、晚段速度 15。箭頭只是視覺，數字本身不分前後段大小，需要由閱讀者判讀：

- 早段高、晚段低 = 起跑快但後段沒勁
- 早段低、晚段高 = 後段衝得起來
- eased / 速度未達標 = 顯示 `–`（en-dash）

---

## 9. Prior Trainer Separator Row `.pp-prior-row`

PP 表中的**跨欄分隔列**，仿 DRF 排版。

### 何時渲染

僅在馬曾經換過訓練師時出現。出現位置在「前任訓練師期間 PP rows」**正上方**。

### 視覺

- 淡黃底 `#fffbe6`
- 上下虛線邊（黃色）
- 黃色徽章 `PREV TRAINER`
- 文字格式：`Previously trained by [名字] [年]: ([starts] sts · [W-P-S] · Win [%])`

### 多次換廄

若有多任前訓練師，會出現多條分隔列，由近到遠依序。

### 後端

陣列 `entry.prior_trainers[]`，每元素：
```json
{
  "name": "Schultz Lindsay",
  "year": "2025",
  "starts": 289,
  "wins": 34,
  "places": 36,
  "shows": 38,
  "win_rate": 0.12,
  "applies_to_pp_indices": [0, 1]  // 哪幾筆 PP rows 是這位訓練師期間
}
```

前端依 `applies_to_pp_indices` 在對應 row 上方插入分隔列。

> **注意**：prototype HTML 中 #1 Ethereal Quality 沒有換廄紀錄，但為了讓設計師看到視覺，目前在 PP rows 中間插了一條 `[demo row]` 標註的示範列，**正式版該馬無換廄就不渲染**。

---

## 10. Works Line `.works-line`

最近練習紀錄，單行文字串。

格式：`[日期] [馬場] [距離] [場地] [時間][符號] [排名]/[總馬數]`

範例：`27Apr26 SA 4f fst :49H 10/18`

- `H` 後綴 = handily（鬆勒慢練）
- `B` 後綴 = breezing（中等強度）
- `Hg` = handily out of gate（出閘練）
- `10/18` = 該日同距離操作中排名第 10、共 18 匹

**後端**：`entry.works[]`，每筆含 date / track / distance / surface / time / handling / ranking / total。

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

### 三態 + 強弱卡片狀態（jky / trn 共用）

```
.normal  ──→ 淡綠底，teal 字
.hot     ──→ 淡橘底，橘字 + ★ 星號
.warn    ──→ 淡紅底，紅字
```

判斷邏輯由後端決定 status 字串，前端只負責 class 對應。

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

最低限度需要的 API response 結構（簡化版）：

```jsonc
{
  "race": {
    "track_name": "Santa Anita",
    "race_number": 9,
    "date": "2026-05-02",
    "post_time": "17:15",
    "timezone": "PT",
    "distance": "1 Mile",
    "surface": "Turf",
    "turns": 1,
    "class_name": "MAIDEN SPECIAL WEIGHT",
    "purse_main": 65000,
    "purse_supplemental": 19500,
    "supplemental_label": "CBOIF",
    "eligibility_text": "Maidens, Fillies 3YO",
    "assigned_weight": 122,
    "beyer_par": 74,
    "track_record_time": "1:31.6",
    "wagers": [...],
    "ai_preview_text": "...",
    "gallop_top3": [...],
    "gallop_value_bet": {...}
  },
  "entries": [
    {
      "post_position": 1,
      "horse_name": "Ethereal Quality",
      "color_sex_age": "B.f.3(Mar)",
      "weight": 122,
      "lasix": true,
      "owner": "Manzanita Stables LLC",
      "morning_line": "15-1",
      "silks_description": "Orange, Yellow Inverted Chevrons, Yellow",
      "silks_pattern": {...},   // 設計師可能想要這個來繪製真實 silks
      "is_home_bred": true,
      "sales_record": null,
      "equipment_change": null,
      "gallop_pace_early": 70,
      "gallop_pace_late": 57,
      "running_style": "Closer",
      "sire": {...}, "dam": {...}, "breeder": {...},
      "jockey": {...},          // name + season + situational stats
      "trainer": {...},         // name + season + 6 個 situational stats
      "lifetime_stats": [...],
      "surface_stats": [...],
      "ai_summary_text": "...",
      "prior_trainers": [],
      "pp_rows": [...],         // 詳見 PP Table
      "works": [...]
    }
  ]
}
```

各欄位細節對應上面各區塊的「後端欄位」欄。
