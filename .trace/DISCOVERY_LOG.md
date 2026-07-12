# DISCOVERY_LOG — Mini Taiwan Pulse 探索紀錄與待解問題

> 產出時間：2026-07-12 · 彙整來源：`.trace/_context/{recon,web_findings,entry_points,data_flow,core_logic,extensions,integrations,configuration}.md`
> 本文件**不重新分析程式碼**，僅去重彙整既有 8 份 Stage 1/1.5/2 文件中標記的 ⚠️ 未驗證項目、文件與程式碼落差、TODO、技術債、待調查區域。

---

## 1. Web Search 發現摘要

`web_findings.md` 因本專案是私有內部專案（無公開官網 / Wiki），搜尋範圍刻意控制在 2 次，聚焦「無法從程式碼本身推得的外部限制」：

- **TDX 運輸資料流通服務**：官方限制為每來源 IP 每秒最多 50 次呼叫；訪客帳號每日 20 次、會員每月約 3,000 次（[TDX 入口](https://tdx.transportdata.tw/)、[Sample Code](https://github.com/tdxmotc/SampleCode)、[介接指南](https://bookdown.org/chiajungyeh/TDX_Guide/)）。但 `integrations.md` 已用程式碼證實**本 repo 前端完全不直接呼叫 TDX**（`grep` 全 `src/` 無 `tdx.transportdata.tw` 字串），TDX 串接邏輯在姊妹 repo `data-collectors`，故此限制與本 repo 前端無直接關係，僅供理解上游資料節奏。
- **Supabase timeout / pg_cron**：[Timeouts 官方文件](https://supabase.com/docs/guides/database/postgres/timeouts) 佐證全域 timeout 2 分鐘，與 CLAUDE.md「Supabase pooler 強制 2min timeout 不能繞，只有 pg_cron 例外」一致；[pg_cron debugging guide](https://supabase.com/docs/guides/troubleshooting/pgcron-debugging-guide-n1KTaz) 提到 pg_cron 最多 32 個並行 job、各佔一條 DB connection，對應 `.claude/memory/PRINCIPLES.md` 「cron 錯開分鐘」的教訓來源。
- **Mapbox GL JS + Three.js**：確認 `CustomLayerInterface` 是兩者共存於同一 WebGL context 的官方支援機制，但未進一步搜尋版本細節，理由是專案內部 `.claude/pitfalls/` 與 `PRINCIPLES.md`（「一個 Mapbox gl context 只掛一個 Three.js CustomLayer 實例」）已記錄專案特定教訓，優先度高於通用網路資料。

---

## 2. 既有文件與程式碼落差清單

| # | 文件說 | 程式碼實際是 | 位置 | 來源 |
|---|---|---|---|---|
| D1 | CLAUDE.md「環境變數」表格與 `docs/launch/03_DEPLOY_RUNBOOK.md:100` 記載 `VITE_DATA_SOURCE=supabase` 用途為「啟用 Supabase（否則用 Pulse API）」 | `grep -rn "VITE_DATA_SOURCE" src/` **無任何結果**；`src/vite-env.d.ts` 的 `ImportMetaEnv` 未宣告此欄位；`nginx.conf` 註解明講「已移除 `/api/` → pulse-api proxy，前端全走 Supabase」 | `CLAUDE.md`、`docs/launch/03_DEPLOY_RUNBOOK.md:100`、`src/vite-env.d.ts` | configuration.md §1.3 |
| D2 | CLAUDE.md 環境變數表列出 `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` / `SUPABASE_SERVICE_ROLE_KEY` / `SUPABASE_DB_URL` | `.env.example` **完全未列**上述四項（僅有 `VITE_MAPBOX_TOKEN`、`VITE_WASTE_MATCHED_TRAILS`、`FR24_API_TOKEN`、S3 四項），是唯一 checked-in 的變數樣板卻不完整 | `.env.example:1-14` | configuration.md §1.1、§1.4 |
| D3 | `docs/launch/03_DEPLOY_RUNBOOK.md:98-101` 把 `VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY` 列為 Zeabur「**Runtime env**」 | Vite 只在 **build time** inline `VITE_*` 前綴變數；`Dockerfile:9-10` 只為 `VITE_MAPBOX_TOKEN` 宣告 `ARG`/`ENV`，**未**為 Supabase 兩變數宣告對應 `ARG`，若這兩變數只在 runtime 存在，build 出的 bundle 會把它們 inline 成 `undefined`，導致 `supabaseConfigured=false` 落回 `createStubClient()` | `Dockerfile:9-13`、`docs/launch/03_DEPLOY_RUNBOOK.md:98-101` | configuration.md §3.1 |
| D4 | `docs/launch/03_DEPLOY_RUNBOOK.md:85` STEP 6 範例指令為 `docker build --build-arg VITE_MAPBOX_TOKEN="<token>" -t pulse-local .` | 該指令完全沒帶 `VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY`，且 Dockerfile 沒有對應 `ARG` 接住，照文件範例操作會建出「Supabase 未設定」的 stub 版本 image | `docs/launch/03_DEPLOY_RUNBOOK.md:85` | configuration.md §3.1 |
| D5 | CLAUDE.md §3「資料載入必須有 Loading UI ⚠️」要求「所有 Supabase 非同步載入都必須註冊 loading task」「禁止靜默 `supabase.rpc().then()`」 | `withLoading()` 的強制套用**沒有測試守備**——`layerConsistency.test.ts` 不掃 loader 檔，只能靠 code review 抓；`extensions.md` 明確標注這是「唯一一個沒有測試守備、只能靠 code review 抓的規則」 | `src/lib/loadingRegistry.ts`、`extensions.md` §2 步驟 2 | extensions.md |
| D6 | CLAUDE.md §5a 鐵則②「分類 ≥ 2 種 → 必寫圖例」，`freewayCongestion` 的 `CONGESTION_COLORS` 有 6 種分類 | `freewayCongestion` 圖例**沒有**——`layerConsistency.test.ts:76` 的 `BASELINE_NO_LEGEND` 集合明確列了 `"freewayCongestion"`，是測試 ratchet 機制已知並「凍結」的缺口，而非漏接；但這與四鐵則規則本身形成對照 | `layerConsistency.test.ts:76`、`useFreewayLayer.ts` | data_flow.md §5.3、extensions.md §5.3 |
| D7 | CLAUDE.md §5a 鐵則③「可選取物件 → 必接 click popup」 | `freewayCongestion` 的 click popup **完全未接線**（`useMapInteraction.ts` 搜尋 `freeway` 無結果），且**未見於任何 baseline 清單登記**——不像圖例缺口有意識地被記錄，popup 缺失可能是純粹遺漏，也可能是刻意設計（線段本身已有顏色/寬度傳達壅塞資訊），本次 trace 未找到明確決策紀錄佐證 | `src/hooks/useMapInteraction.ts`、`src/hooks/useFreewayLayer.ts` | data_flow.md §5.3、§8 |
| D8 | CI（`.github/workflows/ci.yml`）Node 版本設定 | CI 用 `node-version: '20'`，但生產 `Dockerfile:2` 用 `node:22-alpine` build image——Node major 版本不一致（20 vs 22），CI 綠燈不保證生產環境行為完全一致 | `.github/workflows/ci.yml`、`Dockerfile:2` | configuration.md §6.1 |
| D9 | CI 執行 `npm run build`（`tsc -b && vite build`）驗證前端正常性 | CI workflow 全文搜尋不到任何 `env:` 區塊或 `secrets.` 引用，代表所有 `VITE_MAPBOX_TOKEN`/`VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY` 在 CI build 時都是 `undefined`——CI 綠燈只保證「型別正確 + build 不崩潰 + 純邏輯單元測試通過」，**不驗證**金鑰實際可用或地圖/資料能否正常渲染 | `.github/workflows/ci.yml` | configuration.md §6.1 |
| D10 | `src/lib/supabase.ts:3-4` 對 `SUPABASE_URL`/`SUPABASE_ANON_KEY`（無 `VITE_` 前綴）做 fallback 讀取 | 因 Vite 只在編譯期 inline `VITE_*` 前綴變數，無前綴版本在瀏覽器 build 下**永遠是 `undefined`**（不是 runtime 讀不到，是編譯期就被替換掉）——這行 fallback 是死路徑，僅在型別上留了彈性 | `src/lib/supabase.ts:3-4` | configuration.md §1.2 |
| D11 | `layerCatalog.ts` 是 `LayerSidebar`（手機版）與 `IconRailSidebar`（桌機版）共用的**單一真實來源**（資料層） | 但兩者的**渲染邏輯**（含鐵則 4 的 dropdown 閾值 `ctrl.options.length > 3`）在兩個檔案各自重複實作一份完整版本，未抽成共用元件，也**沒有測試守備兩份渲染邏輯是否等價**——ratchet 測試只守備「資料是否存在」，不守備「兩份渲染程式碼是否同步」 | `src/components/LayerSidebar.tsx:530-604`、`src/components/IconRailSidebar.tsx:1250-1288` | extensions.md §6 |

---

## 3. TODO / FIXME / HACK 彙整

`grep -rn "TODO\|FIXME\|HACK" src/ --include=*.ts --include=*.tsx | wc -l` 結果為 **3**（規模極小，全列）：

1. `src/App.tsx:1928` — 註解型 TODO：「Energy MVP：供電燈號 HUD 已搬 monitor 面板（v1.5 TODO）」，屬於已完成搬遷、殘留註解，非待辦阻塞項。
2. `src/chat/types.ts:2` — 檔頭約定型註解：「由 orchestrator 定稿：兩邊各自實作，不要擅改此檔；發現契約不足先在此檔加註 TODO」，是**協作規範**而非具體待辦。
3. `src/map/overlayRegistry.ts:3091` — 具體技術債：「TODO: 3 衍生 layer 的 source URL 尚未產出（D1-D3 ETL pipeline）」——這是唯一一條指向「上游 pipeline 未完成、下游 source URL 待補」的實質性 TODO，值得追蹤是否已在姊妹 repo `taipei-gis-analytics` 有對應進度。

---

## 4. 各 _context 檔案彙整的「⚠️ 未驗證」項目

### entry_points.md（4 項，原文列於文件第 6 節）

1. `useTimeline.ts` 內部驅動 `timeStore.setTime()` 的具體時機（是否含自動播放 RAF、使用者拖曳 timeline 的節流方式）——僅依 `timeStore.ts` 註解與呼叫慣例推斷，未讀該檔案原始碼。
2. `loadingSteps`（`App.tsx:1500` 起）與 `loadingRegistry.snapshot()` 兩套 loading 機制的實際關係（是否有一方最終彙整成另一方）未逐行追蹤，僅由旁證推斷為互補但獨立的兩套。
3. `vite.config.ts:33-37` 的 `/api` proxy 是否仍在生產環境路徑上被使用（`VITE_DATA_SOURCE=supabase` 是否已完全取代 `pulse-api`）未查證——**已在 configuration.md 部分回答**：`VITE_DATA_SOURCE` 是死配置、`nginx.conf` 已移除 `/api/` proxy，但 `vite.config.ts` 的 dev-only `/api` proxy 設定本身是否仍留在檔案中未被移除，仍待確認。
4. `App.tsx` 完整 JSX 樹（2870 行）僅摘要主結構，中段大量 `useEffect`（sessionTracker、mapInteraction、featureInfo 等）未逐一列出，僅做啟動/圖層接線相關的代表性抽樣。

### data_flow.md

- `cachedByKey` 完整實作（LRU 上限邏輯）未在本次 trace 中完整讀取，僅讀取檔首 `cachedOnce` 部分（`loaderCache.ts:1-49`）；`cachedByKey` 簽章從呼叫端可推斷為 `(fetcher, ttlMs, max?) => (key) => Promise<T>`，未逐行核實。
- `timeStore.subscribeDate` 背後的 `dateNotifier`（`src/state/dateNotifier.ts`）本次未展開讀取，僅由 `timeStore.ts:52-53` 註解推斷其 leading+trailing debounce（quietMs 300ms）行為。
- freewayCongestion click popup 缺失是否為刻意設計或純遺漏——未找到明確決策紀錄佐證（見落差表 D7）。

### core_logic.md

- 本文件本身未列獨立「未驗證」章節，但引用了 data_flow.md/entry_points.md 的既有未驗證項目（`timeStore` 相關），並在 §6 freeway vs parking 對照表中指出：freeway 的 popup「未在 `useMapInteraction.ts` 的 `GIS_LAYERS` 表中找到 `road-congestion-hit` 以外的 freeway 專屬項⋯本文件未深入 popup 細節，僅指出 freeway 圖層存在但未走 `PANEL_REGISTRY` 常見的 layerType 命名」。

### extensions.md

- `useTransportParams.ts`/`LegendPanel.tsx` 的 ratchet 測試比對邏輯是**文字比對，非 AST 解析**——若未來改寫這兩檔的程式碼結構，比對邏輯需同步更新，屬於「已知但接受的測試脆弱點」，非傳統意義的未驗證項目，但值得留意。
- `PANEL_REGISTRY` 為何拆成獨立測試檔（`featureInfo/__tests__/registry.test.ts`）而非合併進 `layerConsistency.test.ts`——本文件僅「推測是因為 popup 的全集來源與 layer 全集不是 1:1」，未找到明確設計文件佐證。

### integrations.md

- `styleUrl`（Mapbox map 初始化參數）由外部傳入，未在本次追蹤範圍內展開來源，推測為 Mapbox style URL 或本地自訂 style JSON，屬 `cameraPresets.ts`/呼叫端決定，未實際核對。
- `src/map/overlayRegistry.ts` 與各 `*LayerFactory.ts` 中逐一的 PMTiles source id / layer id 對照表，屬「Layer 系統」而非「外部整合」範疇，本次未逐一列出每個 source id。
- `src/lib/auth.ts`（Supabase Auth/OAuth 整合）與 `src/lib/keyVault.ts`（BYOK API key client-side 儲存機制）皆點名「與外部整合相關但未展開實作細節」。
- Zeabur 平台 build pipeline 是否真的把「Runtime env」透傳進 build container 的 process env（配置章節 D3 的核心假設）——只是「解讀」，未實際驗證 Zeabur 內部行為，需與維護者確認實際部署是否正常運作佐證此假設成立。

### configuration.md

- 同上 D3/D4：Zeabur 是否對未顯式 `ARG` 宣告的變數仍會透傳進 build container，是**假設而非已驗證事實**，文件明確標注「這是本次追蹤中最值得留意的設定脆弱點」。
- `vite.config.ts` 的 `/api` proxy dev-only 設定與 `VITE_DATA_SOURCE` 死配置的關聯（見 entry_points.md 未驗證項 3）。

---

## 5. 已知技術債

1. **雙 Sidebar 渲染邏輯重複但無測試守備兩者等價**（extensions.md §6.3）：`IconRailSidebar.tsx` 與 `LayerSidebar.tsx` 共用 `layerCatalog.ts` 的資料層（`LAYER_COLORS`/`THEMES`/`SECTIONS`），但各自重複實作完整的渲染邏輯（含鐵則 4 的 dropdown 閾值判斷），日後若要改動渲染規則（例如 `> 3` 改成 `> 4`）需同步改兩處，且無自動化機制偵測兩者是否已經不同步。專案在 CLAUDE.md §3「Surgical Changes」原則下刻意容忍此重複，換取資料層絕對一致的簡單心智模型。
2. **`App.tsx` 2870 行單一巨型元件未拆分**（entry_points.md §1.3 觀察）：91 個 hook、38 個 CustomLayer 的接線全部在這一個函式元件內完成，是「Layer-plugin 慣例架構」刻意接受的線性接線成本（extensions.md §2 步驟 6：「接線量隨 layer 數量線性成長，這是本專案有意識接受的權衡」），非傳統意義的「該重構但沒重構」的債務，但對新人 onboarding 與程式碼導覽仍構成負擔。
3. **`freewayCongestion` 圖例缺口已登記但 popup 缺口未登記**（見落差表 D6/D7）：測試 ratchet 機制設計上允許「有意識地不做」（並要求附理由），但目前存在一條規則缺口完全未被任何 baseline 白名單捕捉，顯示規則執行仍有縫隙——這本身也是一種「文件化的例外機制本身不完整」的技術債。
4. **`layerConsistency.test.ts` 用原始碼文字掃描而非 AST 解析**（extensions.md §5.1、configuration.md §5.1）：檔頭註解自承是 heuristic，若 `useTransportParams.ts`/`LegendPanel.tsx` 結構重寫，比對邏輯需要同步更新，否則測試會產生假陰性或假陽性。
5. **`.env.example` 長期落後於實際所需環境變數**（見落差表 D2）：目前唯一 checked-in 的變數樣板缺 Supabase 核心三項，新開發者依此檔設定環境會直接落入 stub client 模式且不易察覺（App 不會 crash，只是整站無資料）。
6. **CI 與生產容器 Node 版本不一致**（D8）：`ci.yml` 用 Node 20、`Dockerfile` 用 `node:22-alpine`，存在版本飄移風險但目前無已知具體故障案例。

---

## 6. 需要更深入調查的區域

1. **`src/hooks/useTimeline.ts` 內部實作**——8 份 context 文件全部只從呼叫慣例與註解推斷其行為（是否含自動播放 RAF、拖曳節流方式），從未真正讀取原始碼，是本次追蹤中被引用次數最多的「未讀檔案」。
2. **`vite.config.ts:33-37` 的 `/api` proxy 是否仍留在程式碼中、是否為死配置殘留**——`nginx.conf` 已證實生產環境不再有 `/api/` → pulse-api 的反向代理，但 dev-only 的 Vite proxy 設定本身是否也該一併清理未被查證。
3. **`VITE_DATA_SOURCE` 是否該從 CLAUDE.md、`docs/launch/03_DEPLOY_RUNBOOK.md`、Zeabur 後台環境變數三處一併移除**——已確認是死配置（§2 D1），但移除範圍與時機需與維護者確認。
4. **Zeabur build pipeline 對「Runtime env 是否透傳進 build container process env」的實際行為**——configuration.md 的整套設定載入優先順序推論建立在此假設上，若假設不成立，代表生產環境長期依賴一個未被正式驗證的隱性機制。
5. **`src/state/dateNotifier.ts`（leading+trailing debounce 實作）**——多處文件引用其行為（quietMs 300ms）但未實際讀取原始碼確認。
6. **`src/lib/loaderCache.ts` 的 `cachedByKey` 完整實作**（LRU 上限邏輯）——僅讀取檔首片段，簽章與行為系推斷而非核實。
7. **`src/lib/auth.ts`（Supabase Auth/OAuth）與 `src/lib/keyVault.ts`（BYOK 金鑰存放）**——兩者皆被多份文件標注為「與安全/整合高度相關但本次未展開」，值得獨立 trace（尤其 `keyVault.ts` 涉及使用者 BYOK LLM 金鑰的 client-side 儲存安全性）。
8. **`freewayCongestion` click popup 缺失的決策脈絡**——需要人工確認（見待確認清單 Q3），無法純靠程式碼追溯。

---

## 7. 建議與維護者確認的問題清單

1. **`.env.example` 是否該補齊 `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` / `SUPABASE_SERVICE_ROLE_KEY` / `SUPABASE_DB_URL`？** 目前新開發者依現有樣板設定會直接落入 Supabase stub client 模式且不易察覺（不 crash，只是無資料）。
2. **`Dockerfile` 該加 `ARG VITE_SUPABASE_URL` / `ARG VITE_SUPABASE_ANON_KEY` 嗎？** 目前生產環境能動是因為「假設」Zeabur 平台透傳 Runtime env 進 build container，若此假設不成立或未來改用純本地 `docker build`（如 runbook STEP 6 範例指令），會靜默建出無資料的 stub 版本 image。
3. **`VITE_DATA_SOURCE` 這個死配置是否該從 CLAUDE.md、`docs/launch/03_DEPLOY_RUNBOOK.md`、Zeabur 後台環境變數三處一併清除？** 程式碼中已無任何消費點，pulse-api 備援分支已在某次重構整個移除。
4. **`freewayCongestion` 圖層缺少 click popup 是刻意設計（線段顏色/寬度已傳達壅塞資訊）還是遺漏？** 若是刻意設計，建議仿照圖例缺口的作法，在對應的 ratchet 測試 baseline（或至少程式碼註解）中明確登記理由，避免日後被誤判為 bug 而重工，也避免規則執行的縫隙持續存在。
5. **CI（Node 20）與生產 Dockerfile（Node 22）版本不一致是否需要對齊？** 若目前無已知故障案例可維持現狀，但建議至少在 `docs/known-issues.md` 或 `.claude/memory/PRINCIPLES.md` 留一筆記錄，避免未來排查詭異行為差異時遺漏這條線索。
