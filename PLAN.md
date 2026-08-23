# 國旗快答（`pg-flagquiz`）— 遊戲規劃文檔

> **用途：** 本 repo 的遊戲權威規格——coding agent 改動前必讀：這個遊戲是什麼、規則、設計限制、優化方向。
> **整理方式：** 從本 repo 實作反向整理（2026-08-23）。**改玩法先改此檔再改碼**；本檔與程式碼衝突時，以「規則（§3）」描述的設計意圖為準回報差異。
> **上游契約：** [PG-GAME-AGENT-GUIDE.md](https://github.com/sampot/playgrounds/blob/main/docs/PG-GAME-AGENT-GUIDE.md)（唯一必讀；本檔不重複其全文）· 型錄條目 `playgrounds/catalog/entries/pg-flagquiz.yaml`

## 1. 一句話

看國旗從四個國名裡搶答的 60 秒限時測驗——連擊與剩餘秒數加成分數，挑戰 KV 最高紀錄；題庫 46 國含臺灣，致敬問答競速類型。

## 2. 定案速覽

| 項 | 值 |
| --- | --- |
| catalog id / kind / series | `pg-flagquiz` / `game` / `懷舊` |
| status | `listed` |
| 模式 | 單人 60 秒限時挑戰；單一題型「旗→國名」四選一；無關卡 |
| 題庫 | `FLAGS` 46 國（含 `TW` 臺灣）；對應 `assets/flags/*.png` 46 檔 |
| 計分 | 答對 `100 + combo×10 + max(0, floor(remaining))×5`；答錯不清分、只歸零連擊 |
| 紀錄 | `/api/kv/pg-flagquiz-best`（localStorage 快取＋KV 權威） |
| 素材 | Kenney Flag Pack PNG（CC0）；**無音效、無震動** |
| 交付形 | 純 HTML＋CSS＋ESM JS；無 build；`npx vitest run` 測試 |

## 3. 完整規則（現行實作）

### 3.1 局流程

- 按「開始 60 秒挑戰」→ 重置 state（`newGame()`：score/combo/answered/correct 全 0）、`time = 60`、`active = true`；以 `setInterval` 每 1000ms 遞減。
- 出題不限時：答一題立即換下一題（`pick` → `next()`），直到 `time <= 0` → `clearInterval`、`active = false`、顯示「時間到！答對 X／Y」並 `save()`。
- `active === false` 時任何點選無效（`pick` 直接 return）；挑戰進行中開始鈕 disabled。重按開始會先清掉舊 timer。

### 3.2 出題（`makeQuestion(rng)`）

- 正解：`FLAGS[floor(rng×46)]` 隨機取一。
- 干擾項：其餘 45 國經 `sort(() => rng() - .5)` 洗牌後取前 3——注意此洗牌法分布不均（部分排列機率偏低），僅影響干擾項抽樣公平性，功能正確。
- 四選項 `[flag, ...others]` 再同法排序；同一題內 code 必不重複（filter 保證），且必含正解。
- `rng` 可注入（測試用固定值重放）。

### 3.3 計分（`answer(state, q, code, remaining)`）

- 答對：`combo += 1`、`correct += 1`、`score += 100 + combo×10 + max(0, floor(remaining))×5`。首題秒答例：combo 變 1、remaining=60 → **410 分**。
- 答錯：`answered += 1`、`combo = 0`；分數不扣。
- UI 於每次作答後顯示「答對！〈國名〉」或「答案是〈國名〉」（教學回饋）。

### 3.4 紀錄與邊界

- 載入：先讀 localStorage `pg-flagquiz-best`，再非同步 GET `/api/kv/pg-flagquiz-best` 取兩者 max（KV 失敗靜默退 LS）。
- 寫入：僅當本局分數**嚴格大於** best 才寫 LS＋PUT KV；PUT 失敗吞掉（離線仍留 LS 快取）。
- 時間參數由 app 層傳入當下 `time`；`answer()` 的 remaining 參數預設 0（不帶即無時間加成）。

## 4. 操作與畫面

| 輸入 | 動作 |
| --- | --- |
| 開始鈕 | 開始／（結束後）再來一場 |
| 點國名按鈕 ×4 | 作答（立即換題） |

- HUD 三欄：目前分數、「連擊 N｜剩餘 X 秒」、最高紀錄；`status` 掛 `role="status"`（螢幕閱讀器可及）。
- 旗幟 `<img id="flag">` 的 alt 固定「待辨識國旗」（防國名劇透）；初始預載 `TW.png`。
- `<details>` 收合遊戲說明；footer 指向 ATTRIBUTION.md。Mobile-first 單欄；禁原生對話框。

## 5. 持久化（KV 權威）

| key | 內容 | 讀寫時機 |
| --- | --- | --- |
| `pg-flagquiz-best` | 歷史最高分（字串數字） | 載入時 LS→KV 取 max；破紀錄時 LS 寫入＋KV PUT |

- key 已帶 `pg-<id>-` 前綴，符合宿主 KV 慣例（宿主 KV 無 per-SAM 命名空間，新 key 必須比照前綴化）。
- 分層規則：`/api/kv` 為唯一權威，localStorage 只是離線快取（姊妹作 pg-airhockey 同模式）。
- `functions.js` 為 stub（default export `fetch` 回 ok/name/path JSON），未提供自訂 functions API；現行前端也不經過它。

## 6. 美術／音效／署名

- `assets/flags/`：46 面國旗 PNG — Kenney Vleugels《Flag Pack》（CC0 1.0，https://kenney.nl/assets/flag-pack ）；授權副本 `assets/flags/License.txt`；`ATTRIBUTION.md` 已列。PNG 檔名＝ISO 代碼，與 `FLAGS[].code` 一一對應。
- 無音效資產、無合成音效（全作靜音）。
- 新增／替換旗幟：PNG 拷進 `assets/flags/`、`FLAGS` 加條目、同步 `sam-manifest.json` files、更新 `ATTRIBUTION.md`（CC0 也署名）。

## 7. 測試（`npx vitest run`）

現有覆蓋（`game.test.js`，4 例）：題庫 ≥40 面且含 `TW`；題目恰 4 個不重複選項且含正解；答對加分（>100）且 combo 計 1；答錯 combo 歸零。

UI/timer/save 不在測試範圍。若擴充計分或出題，最小必測建議：`remaining = 0` 時分數不含時間項、best 僅嚴格大於時更新、`FLAGS` 條目與 `assets/flags/*.png` 一一對應（防漏檔白圖）。

## 8. 硬約束（不可違反）

1. 僅 HTML＋CSS＋JS（ESM）；**無 build**、不入庫 `node_modules`、不安套件；工具一律 `npx <pkg>` 臨時執行。
2. 禁瀏覽器原生 `alert`／`confirm`／`prompt`；回饋一律頁內 status 區。
3. Mobile-first；主操作不可 hover-only。
4. 最高分以 `/api/kv/pg-flagquiz-best` 為權威；裸 localStorage 僅可當快取；新增統計 key 一律 `pg-flagquiz-*` 前綴。
5. 不自行載入 `sdk.js`；宿主注入 `window.PG`（本作未用）。
6. 改動可執行邏輯前先寫失敗測試（TDD）；`makeQuestion`/`answer` 的 `rng` 注入介面不得移除。
7. 檔案清單變動須同步 `sam-manifest.json`（46 面 PNG 全數在列）。
8. 內容契約：`FLAGS` 條目 ↔ `assets/flags/<code>.png` 必須一一對應；調整題庫下限（現行測試守 ≥40）須同步改測試並在此登記。

## 9. 優化建議（可玩性與樂趣）

依優先級；實作前先在此登記並補測試。原則：強化節奏感與學習成就感，不改變「看旗搶答」的核心認同。

**高優先**

1. **音效與倒數張力**：全作無聲是最大手感缺口。做：WebAudio 合成答對上行雙音／答錯低鳴／最後 5 秒 tick 聲（比照姊妹作 `audio.js` 的 tone 模式，手勢 unlock）。為何：限時搶答的緊迫感主要靠聽覺驅動。
2. **題型擴充**：只有「旗→國名」一種，熟手 60 秒後疲勞。做：反向題（國名→點四面旗）、大洲限區（`FLAGS` 加 region 欄位只出該區）、干擾項改抽「視覺易混淆旗」（同色系）提升鑑別度。做法：`makeQuestion(mode)` 參數化，TDD 先行。

**中優先**

3. **連擊存在感**：combo 是核心計分卻只在文字列出現。做：combo ≥5 時邊框發光、每 5 連擊 toast；最高連擊併入 KV（單一 JSON key 或 `pg-flagquiz-best-combo`）。為何：給高手可炫耀的次級目標。
4. **錯題重考權重**：session 內記住答錯的旗幟，提高近期重出率（答錯當下已顯示正解，形成閉環學習）。為何：把小遊戲變成有效率的記憶工具，增加「再玩一次會更強」的動機。
5. **生涯戰績**：`/api/kv/pg-flagquiz-stats` 存場次/平均分/最佳，結算面板顯示縱向進步曲線要點。為何：60 秒一局的遊戲需要長線鉤子。

**低優先**

6. **干擾項洗牌均勻化**：`sort(() => rng() - .5)` 分布不均，改 Fisher–Yates（行為不變、抽樣公平，順手讓決定性測試更穩）。
7. **輸入細節**：選項鍵盤快捷 1–4、作答後短暫 lock 防連點誤觸下一題。
