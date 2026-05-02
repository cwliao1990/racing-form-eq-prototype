---
title: Gallop AI PP 資料來源與進階參數推導
audience: shared
language: zh
status: draft
last-updated: 2026-05-01
schema-source: C:\Users\cwlia\Desktop\gallop_schema_output (2026-04-14 版, 36 tables)
related:
  - pp-feature-gap.md (vs DRF Classic 缺口分析)
  - racing-form-eq.html (prototype)
---

# Gallop AI PP 資料來源與進階參數推導

對 prototype `racing-form-eq.html` 上每一個可見資訊，列出它在資料庫裡的來源欄位，以及無法直接取得的進階參數該如何計算。

---

## 圖例

| 圖示 | 含義 |
|---|---|
| ✅ | schema 中已有對應欄位，可直接取用 |
| 🔁 | schema 中有，但需要 join 多表或字串解析才能組出顯示值 |
| ➕ | 目前 schema 中沒有對應，需要從其他來源補（例如資料供應商 feed、額外計算結果） |
| 🤖 | AI / ML 模型輸出，不在交易型 schema 中產生 |

---

## 0. 整體架構

```
Race Header Card
AI Race Preview
Race Analysis（Top 3 Prediction、Value Bet）
Horse Info Card
  ├─ 名字 / 顏色 / Aux meta（Sales、Equip）/ Pace
  ├─ Pedigree（Sire / Dam / Breeder）
  ├─ Jockey、Trainer 名字 + 季統計
  ├─ Lifetime / Year / Surface 統計表
  ├─ Jockey Compact Cards（5 張情境卡）
  ├─ Trainer Compact Cards（6 張情境卡）
  └─ AI Horse Summary
PP Table（21 欄 + Prior Trainer 分隔列）
Works Line
```

---

## A. Schema 對齊全域摘要

### A-1. 直接可取（✅）

| 顯示需求 | 對應欄位 |
|---|---|
| 馬場名、時區 | `track.name`, `track.timezone` |
| 賽事日、開賽時間 | `meeting.raceDate`, `race.raceTime` |
| 比賽序號、距離、地面、級別、年齡/性別限制 | `race.seq`, `race.distance`, `race.courseSurface`, `race.type`, `race.grade`, `race.ageRestriction`, `race.sexRestriction` |
| 主獎金、認領金額 | `race.purse`, `race.claimingPrice` |
| 馬名、毛色、性別、生日、父母、breeder | `horse.name`, `horse.color`, `horse.sex`, `horse.foalingDate`, `horse.sire`, `horse.dam`, `horse.breeder` |
| 出閘號、program number、鞍布顏色、賦重 | `runner.postPosition`, `runner.programNumber`, `runner.saddleClothColor`, `runner.weightCarried` |
| 賽前 / 賽後賠率、裝備、藥物 | `runner.preWinOdds`, `runner.postWinOdds`, `runner.equipment`, `runner.medication` |
| 名次、獎金、賽事評論 | `runner.rank`, `runner.earn`, `runner.comment` |
| 分段時間、分段距離（賽後） | `runner.fractionList` (JSON), `runner.pointOfCallList` (JSON) |
| 騎師 / 訓練師 / 馬主名字 | `jockey.name`, `trainer.name`, `owner.name`（透過註冊號碼 join） |
| 生涯統計（出賽 / 勝 / 獎金） | `career_race_summary.*` |
| 年度 / 場地 / 馬場 / 距離 / 騎師 / 訓練師 切片統計 | `race_summary` 與 `race_summary_by_*` 系列 |

### A-2. 需 join 或解析（🔁）

| 顯示需求 | 推導方式 |
|---|---|
| Eligibility 文字（Maidens, Fillies 3YO） | `race.ageRestriction` + `race.sexRestriction` 字串拼接 |
| Class 顯示文字（Md Sp Wt 72k） | `race.type` + `race.grade` + `race.purse / 1000` |
| Lifetime / Year / Surface 4×6 統計表 | `career_race_summary` 一筆 + `race_summary_by_year` 多筆 + `race_summary_by_course_surface` 多筆組合 |
| Sales 自繁判定 | `horse.breeder == owner.name` 名稱比對為 Home Bred |
| Trip 落後身位（每個 call 點） | `runner.pointOfCallList` JSON 解析（需含 `lengthsBack` 欄位，待確認） |
| PP rows 一筆過往賽歷 | `runner_past_performance.pastRunnerId` 鏈到歷史 `runner` 記錄 → `race` → `meeting` → `track` → `jockey` / `trainer` |
| Prior Trainer 換廄歷史 | 比對該 horse 所有歷史 `runner.trainer` 註冊號碼，找連續變更點，分組對應的 PP rows |
| 換廄區間訓練師當年戰績 | `race_summary_by_trainer` + `race_summary_by_year` 該訓練師當年資料 |
| Equipment change 是否「首次」 | 該 runner 的 `equipment` 與該 horse 上一場 `runner.equipment` 比對 |
| 賦重前綴 L（Lasix） / 後綴 b f（綁帶） | `runner.medication` 是否含 L、`runner.equipment` 是否含 b/f |

### A-3. schema 沒有、需從外部補（➕）

#### A-3a. Race 層級

| 缺項 | 對應 prototype 顯示 |
|---|---|
| 該級別標準速度指數（Beyer Par 74） | Race header 中段 |
| 同距離歷史最快（Track Record 1:31³） | Race header 中段 |
| 附加獎金與標籤（+$19,500 CBOIF） | Race header 主獎金後括號 |
| 結構化 Wagers + takeout 百分比 | Race header 第二行 wager 列 |
| 預設賦重（race-level 122 lbs） | Race header 中段 |
| 賽道彎道數（1 turn / 2 turn） | Track diagram 文字 |
| Race Insured 旗標 | Race header 右側徽章 |

#### A-3b. PP Row 層級

| 缺項 | 對應 prototype 顯示 |
|---|---|
| 每場 Beyer Speed Figure | PP `Speed Index` 欄 |
| 每場 TimeformUS Pace E→L | PP `Pace E→L` 欄 |
| Top finishers 結構化（馬名 / 賦重 / 落後身位） | PP `Top Finishers` 欄目前是 text，理想是結構化 |

#### A-3c. 進階情境統計

現有 `race_summary_by_trainer` / `race_summary_by_jockey` 是「按年份」聚合。Prototype 用的是「按比賽情境」聚合，schema 中尚未存在。

**訓練師 6 情境**（PP 上 trainer-card）：

| 情境代號 | 篩選條件 |
|---|---|
| 61-180D | 該訓練師讓馬離場 61~180 天回歸的所有 runner |
| 1stTurf | 該訓練師訓練的馬第一次跑草地的 runner |
| Dirt/Turf | 該訓練師訓練的馬從泥地切換到草地的 runner |
| Turf | 該訓練師所有草地賽 runner |
| Routes | 該訓練師所有 ≥1 mile 賽 runner |
| MdnSpWt | 該訓練師所有 `race.type = "Maiden Special Weight"` 的 runner |

**騎師 5 情境**（PP 上 jockey-card）：

| 情境代號 | 篩選條件 |
|---|---|
| 本年度整體 | 當前年度全部 runner |
| Turf | 該騎師所有草地賽 |
| 1 Mile | 該騎師所有 1 mile 賽 |
| >6/1 LS | 該騎師 `runner.preWinOdds > 6/1` 的 longshot 賽 |
| J/T 搭配 | 該騎師與當前訓練師組合的所有 runner |

每組產出：
```
{
  starts,                     // COUNT(*)
  wins,                       // COUNT(rank=1)
  winRate,                    // wins / starts
  roi2dollar,                 // SUM(rank=1 ? (preWinOdds_decimal+1) * 2 : 0) / (starts * 2)
  status: 'normal'|'hot'|'warn'  // 由 backend 依 winRate / ROI 判斷
}
```

#### A-3d. AI / ML 輸出（🤖）

| 顯示位置 | 模型輸出 |
|---|---|
| AI Race Preview 整段文字 | 整場文字分析 |
| Race Analysis 表 `Speed Index AI` | runner 速度指數 |
| Race Analysis 表 `Style` | runner 跑法分類 |
| Race Analysis 表 `Win% AI` | runner 勝率預測 |
| Race Analysis 表 `Key Note` | runner 30 字內重點 |
| Top 3 Prediction | race 排序預測陣列 |
| Value Bet | race 單一 longshot 預測 |
| Horse Info `Gallop AI Pace` | runner 早段、晚段速度 |
| AI Horse Summary | runner 文字分析 |

#### A-3e. 其他缺項

| 缺項 | 對應 prototype 顯示 |
|---|---|
| 拍賣 / Sales 紀錄 | Aux meta line 的 Sales 欄 |
| Equipment change 與前場比對 | Aux meta line 的 Equip 欄 |
| Workouts 練習紀錄（schema 中無對應表） | Works Line |
| Sire stud fee（如 $100,000） | Pedigree Sire 行括號內 |
| Silks pattern 結構化（不只顏色文字） | Horse Info 鞍布視覺化 |
| 海外賽歷格式（歐洲、香港等用詞單位差異） | PP Table foreign row |

---

## 1. Race Header Card

最上方比賽基本資訊區。

| 顯示欄位 | 範例 | 來源 |
|---|---|---|
| Track | Santa Anita | ✅ `track.name`（`race.meetingId → meeting.code → track.code → track.name`） |
| Race # | R9 | ✅ `race.seq` |
| Date | Sat May 02 | ✅ `meeting.raceDate` |
| Post Time | 5:15 PT | 🔁 `race.raceTime` (UTC) → 依 `track.timezone` 轉換 + 24 小時制顯示 |
| Distance | 1 Mile | ✅ `race.distance` |
| Surface | Turf | ✅ `race.courseSurface` |
| Course Type | Inner | ✅ `race.courseType` |
| Class | MAIDEN SPECIAL WEIGHT | 🔁 `race.type` + `race.grade` 合併 |
| Purse 主獎金 | $65,000 | ✅ `race.purse` |
| Purse 附加 | +$19,500 CBOIF | ➕ |
| Eligibility | Maidens, Fillies 3YO | 🔁 `race.ageRestriction` + `race.sexRestriction` 拼接 |
| 預設賦重 | 122 lbs | ➕（也可能在 `race.raceCondition` text 內，待 parse 確認） |
| Beyer Par | 74 | ➕ |
| Track Record | 1:31³ | ➕ |
| Wagers + takeout | Win $2 ... Late Pick 3 $3 (15% takeout) | ➕（`race.raceCondition` 可能包含 wager 文字，但 takeout % 通常另存） |
| Track Diagram（橢圓 SVG） | 1mT · 1 turn | 🔁 由 `race.distance` + `race.courseSurface` + `track.code` 推算彎道數，目前 `track.turnCount` 不存在屬於 ➕ |
| Race Insured 旗標 | 徽章 | ➕ |

---

## 2. AI Race Preview

整場 AI 文字分析。

| 顯示欄位 | 來源 |
|---|---|
| AI 整場分析文字（HTML 或 Markdown） | 🤖 ➕ |

---

## 3. Race Analysis Section

### 3a. 全場馬匹表

| 欄位 | 範例 | 來源 |
|---|---|---|
| # | 5 | ✅ `runner.postPosition` |
| Horse | Soul Sister | 🔁 `runner.registrationNumber → horse.name` |
| Speed Index AI | 89 | 🤖 ➕ |
| Turf Exp | 5 starts (3 Turf) | 🔁 `race_summary_by_course_surface` 取該馬的 Turf 列，再從 `career_race_summary` 取總出賽數 |
| Style | Midpack | 🤖 ➕ |
| M/L | 5/2 | ✅ `runner.preWinOdds`（**需確認此欄是否就是 morning line**） |
| Win% AI | 28.6% | 🤖 ➕ |
| Key Note | 30 字內文字 | 🤖 ➕ |

### 3b. Top 3 Prediction

| 欄位 | 來源 |
|---|---|
| 排序陣列（1st / 2nd / 3rd 馬號 + 馬名 + 短理由） | 🤖 ➕ |

### 3c. Value Bet

| 欄位 | 來源 |
|---|---|
| 單一馬號 + tagline + 4~5 條理由列表 | 🤖 ➕ |

---

## 4. Horse Info Card

### 4a. 名字區塊

| 欄位 | 範例 | 來源 |
|---|---|---|
| 馬名 | Ethereal Quality | ✅ `horse.name` |
| 毛色 / 性別 / 年齡（出生月） | B.f.3(Mar) | 🔁 `horse.color` + `horse.sex` + `horse.foalingDate` 推算年齡與月份 |
| 賦重 + Lasix 前綴 | L122 | 🔁 `runner.weightCarried` + `runner.medication` 含 L 時前綴 |
| Owner | Manzanita Stables LLC | 🔁 `runner.owner` 註冊號碼 → `owner.name` |
| Morning Line | 15-1 | ✅ `runner.preWinOdds`（待確認語意） |
| 鞍布顏色文字 | Orange, Yellow Inverted Chevrons, Yellow | ✅ `runner.saddleClothColor` |

### 4b. Aux Meta（Sales / Equip）

| 顯示文字 | 推導 |
|---|---|
| Sales: KEESEP24 $210k | ➕ `horse.salesRecord` 暫無，需要拍賣資料來源 |
| Sales: Home Bred (no auction) | 🔁 條件：無拍賣紀錄 且 `horse.breeder == owner.name` |
| Sales: No public sale | 🔁 條件：無拍賣紀錄 且 `horse.breeder != owner.name` |
| Equip: Blinkers ON | 🔁 該場 `runner.equipment` 與該馬上一場 `runner.equipment` 比對 |
| Equip: no change | 🔁 與上一場相同 |

### 4c. Gallop AI Pace

| 欄位 | 來源 |
|---|---|
| Early 數字 | 🤖 ➕ |
| Late 數字 | 🤖 ➕ |
| 中間 Style 文字 | 🤖 ➕ |

### 4d. Pedigree

| 欄位 | 來源 |
|---|---|
| Sire 名字 | ✅ `horse.sire`（為註冊號碼字串） |
| Sire 之 Sire（祖父系） | 🔁 用 sire 註冊號碼再 join `horse` 表取其 sire |
| Sire stud fee（$100,000） | ➕ |
| Dam 名字 | ✅ `horse.dam` |
| Dam 之 Sire（外祖父系） | 🔁 同 sire 邏輯 |
| Breeder | ✅ `horse.breeder` |
| 出生州（Ky） | 🔁 從 `horse.breeder` 字串末段 parse，或新欄位 |

### 4e. 騎師 / 訓練師基本行（Horse Info Card 內）

| 顯示 | 範例 | 來源 |
|---|---|---|
| 騎師名字 | Frey K | 🔁 `runner.jockey → jockey.name` |
| 騎師當前 meet 統計 | (21 2 0 8 .10) | 🔁 `race_summary_by_*` 篩選該 meet（需 meet 維度，現有 by_year / by_track 可組） |
| 騎師年度統計 | 2026: (162 13 .08) | 🔁 `race_summary_by_year` 篩騎師 |
| 訓練師同上 | Drysdale Neil + 兩組數字 | 🔁 同上邏輯 |

### 4f. Lifetime / Year / Surface 統計表

| Row | 來源 |
|---|---|
| Life | ✅ `career_race_summary` 該馬一筆 |
| 2026 / 2025 | ✅ `race_summary_by_year` 該馬多筆 |
| SA（當前馬場） | ✅ `race_summary_by_track` 該馬篩選 |
| D.Fst / Wet / Synth / Turf | ✅ `race_summary_by_course_surface` 該馬多筆 |
| Dst（同距離） | ✅ `race_summary_by_distance` 該馬篩選同距離 |
| 各列右側「最佳速度指數」 | ➕ 各 summary 表目前無此欄 |

---

## 5. Jockey Compact Cards（5 張）

| 卡片 | label | 來源 |
|---|---|---|
| 1 | 2026 | ✅ `race_summary_by_year` 該騎師當年 |
| 2 | Turf | 🔁 動態 SQL：該騎師所有 `race.courseSurface = Turf` 的 runner 聚合 |
| 3 | 1 Mile | 🔁 動態 SQL：該騎師所有 `race.distance = 1 Mile` 的 runner 聚合 |
| 4 | >6/1 LS | ➕ 需 SQL 對 `runner.preWinOdds` 解析數值 + 篩選 > 6 |
| 5 | J/T 搭配 | ➕ 需 SQL 同時篩選 `runner.jockey = X AND runner.trainer = Y` |

每卡產出 `{ starts, wins, winRate, status }`。

ROI 不在 jockey 卡片顯示（騎師卡只顯示 wins + winRate），但若擴充顯示則套用同一公式。

---

## 6. Trainer Compact Cards（6 張）

| 卡片 | label | 篩選條件 | 來源 |
|---|---|---|---|
| 1 | 61-180D | 該訓練師讓馬離場 61~180 天回歸 | ➕ 需計算「該馬上一場到此場間隔天數」並篩入 |
| 2 | 1st Turf | 該訓練師訓練的馬第一次跑草地 | ➕ 需計算「此前該馬有無草地紀錄」 |
| 3 | Dirt/Turf | 該訓練師訓練的馬從泥地切換到草地 | ➕ 需計算「該馬上一場 surface != Turf 且此場 surface = Turf」 |
| 4 | Turf | 該訓練師所有草地賽 | 🔁 動態 SQL（同 jockey Turf 邏輯） |
| 5 | Routes | 該訓練師所有 ≥1 mile 賽 | 🔁 動態 SQL（distance ≥ 1 mile） |
| 6 | MdnSpWt | 該訓練師所有 `race.type = "Maiden Special Weight"` | 🔁 動態 SQL |

每卡產出 `{ starts, wins, winRate, roi2dollar, status }`。

### ROI 公式

```
roi2dollar = SUM( IF(rank=1, (preWinOdds_decimal + 1) × 2, 0) ) / (starts × 2)

其中 preWinOdds_decimal 從 "5/2" 之類的分數轉成 2.5
```

| ROI 值 | 含義 |
|---|---|
| > $2.00 | 正期望值 |
| = $2.00 | 損益兩平 |
| < $2.00 | 負期望值 |

### 卡片 status 三態判斷邏輯

| status | 條件範例 |
|---|---|
| `hot` | ROI > $2.00 且 winRate ≥ 季平均 × 1.3 |
| `warn` | ROI < $1.00 或 winRate ≤ 季平均 × 0.7 |
| `normal` | 其他 |

具體閾值由 backend 視資料分布決定。

---

## 7. AI Horse Summary

| 欄位 | 來源 |
|---|---|
| 文字分析（針對單一馬） | 🤖 ➕ |

---

## 8. PP Table（21 欄）

每筆 PP row 取自 `runner_past_performance.pastRunnerId` 鏈到的歷史 runner 紀錄，需 join：

```
runner_past_performance.runnerId = 當前出賽的 runner.id
runner_past_performance.pastRunnerId = 歷史 runner.id
   → race（過往比賽：距離、地面、級別、獎金、時間）
   → meeting → track（馬場、時區）
   → jockey, trainer（透過註冊號碼）
```

### 21 欄逐欄

| # | Header | 範例 | 來源 |
|---|---|---|---|
| 1 | Date | 8Jan26 | ✅ `meeting.raceDate` |
| 2 | Rc/Trk | 4SA | 🔁 `race.seq` + `track.code` |
| 3 | Dist | 6f / 1 1/16 | ✅ `race.distance` |
| 4 | Sf | fst / fm | 🔁 `race.courseSurface` 需 normalize 為代碼 |
| 5 | Class | Md Sp Wt 72k | 🔁 `race.type` + `race.grade` + `race.purse / 1000` |
| 6 | Cond | C / - | ✅ `race.trackCondition` |
| 7 | PP | 4 | ✅ `runner.postPosition` |
| 8 | 1/4 | :22³ | 🔁 `runner.fractionList` JSON 第 1 段 |
| 9 | 1/2 | :45⁴ | 🔁 `runner.fractionList` JSON 第 2 段 |
| 10 | 3/4 | :58 | 🔁 `runner.fractionList` JSON 第 3 段 |
| 11 | Str | 7¹¾ | 🔁 `runner.pointOfCallList` JSON 倒數第二點（位置 + lengthsBack） |
| 12 | Fin | 4²¾ | 🔁 `runner.pointOfCallList` JSON 最末點 |
| 13 | Jockey | Berrios H | 🔁 `runner.jockey → jockey.name` |
| 14 | Wt | L122f | 🔁 `runner.weightCarried` + `runner.medication`(L) + `runner.equipment`(b/f) |
| 15 | Odds | 48.60 | ✅ `runner.postWinOdds`（無則 fallback `preWinOdds`） |
| 16 | #Rnrs | 10 | ✅ `race.numberOfRunners` |
| 17 | Time | 1:11² | 🔁 `runner.fractionList` 最末段 |
| 18 | Speed Index | 58 | ➕ |
| 19 | Pace E→L | 81→15 | ➕ |
| 20 | Purse | $72k | ✅ `race.purse / 1000` |
| 21 | Top Finishers + Comment | 文字 | 🔁 `runner.comment` 含 trip comment；top finishers 結構化資料 ➕ |

### Wt 欄藥物 / 裝備標記推導

| 顯示字元 | 來源條件 |
|---|---|
| `L` (前綴) | `runner.medication` 包含 'L' |
| `b` (後綴) | `runner.equipment` 包含 'b' |
| `f` (後綴) | `runner.equipment` 包含 'f' |

`runner.medication` 與 `runner.equipment` 為 varchar，**儲存格式（CSV / 代碼 / 完整文字）需確認**。

### Str / Fin 欄落後身位上標推導

從 `runner.pointOfCallList` JSON 解析。理想結構：

```json
[
  { "call": "start", "position": 8, "lengthsBack": null },
  { "call": "1/4",   "position": 8, "lengthsBack": 2.75 },
  { "call": "1/2",   "position": 8, "lengthsBack": 2.75 },
  { "call": "str",   "position": 7, "lengthsBack": 1.75 },
  { "call": "fin",   "position": 4, "lengthsBack": 2.75 }
]
```

**現有 JSON 是否含 `lengthsBack` 欄位待確認**。若不含，需從 Equibase / DRF trip line 補。

`eased`、`pulled up` 等非完賽情況顯示文字而非身位數字。

---

## 9. Prior Trainer Separator Row（PP 表內換廄分隔列）

### 換廄偵測邏輯

```
SELECT runner.trainer, race.raceTime
FROM runner JOIN race ON runner.raceId = race.id
WHERE runner.registrationNumber = <該馬>
ORDER BY race.raceTime DESC
```

掃描 trainer 連續變更點，**每次 trainer 註冊號碼變化 = 一個換廄事件**。每個換廄事件對應的 PP rows = 「該訓練師期間的所有歷史 runner」。

### 顯示資料

| 欄位 | 來源 |
|---|---|
| 訓練師名字 | 🔁 `trainer.name`（用變更前的註冊號碼 join） |
| 該訓練師當年戰績（289 sts · 34-36-38 · Win 12%） | 🔁 `race_summary_by_trainer` + `race_summary_by_year` 該訓練師當年聚合 |
| 此分隔列覆蓋哪些 PP rows | 🔁 由換廄區間決定 |

---

## 10. Works Line

| 顯示 | 來源 |
|---|---|
| 整段 workouts 紀錄 | ➕ schema 中無對應表 |

每筆 workout 包含 date / track / distance / surface / time / handlingCode / ranking / total。

handlingCode 含義：

| 字元 | 含義 |
|---|---|
| H | handily（鬆勒慢練） |
| B | breezing（中等強度） |
| Hg | handily out of gate（出閘練） |

---

## 11. 進階參數計算邏輯彙整

歸納 prototype 上所有「不能直接 SELECT、需要計算」的數值，集中列出來源演算法。

### 11a. 換廄偵測

> 已於第 9 節描述。

### 11b. 賦重前後綴解析

> 已於第 8 節 Wt 欄描述。

### 11c. 落後身位

> 已於第 8 節 Str / Fin 欄描述。

### 11d. 自繁判定（Sales 空值文字決策）

```
if horse.salesRecord exists:
    text = format(horse.salesRecord)         // 例如 KEESEP24 $210k
elif horse.breeder == owner.name:
    text = "Home Bred (no auction)"
else:
    text = "No public sale"
```

### 11e. Equipment change 與前場比對

```
prevRunner = 該 horse 上一場 runner 紀錄（依 race.raceTime DESC 取第二筆）
if currentRunner.equipment != prevRunner.equipment:
    if currentRunner.equipment == "Blinkers":
        text = "Blinkers ON"
    elif prevRunner.equipment == "Blinkers" and currentRunner.equipment != "Blinkers":
        text = "Blinkers OFF"
    else:
        text = format diff
else:
    text = "no change"
```

### 11f. 訓練師 / 騎師情境統計

> 已於第 5、6 節描述每個情境的篩選條件與聚合公式。

### 11g. ROI 計算

```
roi2dollar = SUM( IF(rank=1, (oddsToDecimal(preWinOdds) + 1) × 2, 0) )
             /
             (starts × 2)

oddsToDecimal("5/2") = 5/2 = 2.5
oddsToDecimal("15-1") = 15
oddsToDecimal("*1.50")（裁判賠率）= 1.50
```

### 11h. 卡片狀態 hot / warn 判斷

> 已於第 6 節描述閾值範例，實際值由資料分布調整。

### 11i. PP rows 取得鏈

> 已於第 8 節描述 join 路徑。

### 11j. 海外賽歷格式

歐洲賽歷（如 #9 Nerida 在 Toulouse）有以下差異：

| 屬性 | 北美 | 歐洲 |
|---|---|---|
| 距離單位 | furlongs | metres |
| 彎道方向 | 預設逆時針，少數順時針 | 標 RH（順）/ LH（逆） |
| 場地代碼 | fst / fm | gd（good）/ sf（soft）等 |
| 級別命名 | Md Sp Wt / Alw / Stk | Listed / Group 1~3 / Conditions |

schema 是否已支援這些變體待確認。

---

## 12. 待確認事項

1. `runner.preWinOdds` 是 morning line（晨盤）還是賽前最後賠率？兩者語意不同。
2. `runner.fractionList` / `runner.pointOfCallList` JSON 的內部 schema 確切是什麼？是否含 `lengthsBack`？
3. `runner.medication` 與 `runner.equipment` 的儲存格式（CSV / 代碼 / 完整文字）。
4. TimeformUS / Beyer 是否已有授權或計畫採用？兩者皆為第三方資料。
5. AI 模型目前狀態：是已可呼叫的 service，還是待建？輸出 schema 待定。
6. 資料供應商來源（Equibase / DRF / 自有 scraper）會影響 Sales、Workouts、Beyer 取得方式。
7. 海外賽歷（歐洲、香港、日本）佔比，影響海外 row 格式優先級。
