# Stage 1 偵察報告 — Mini Taiwan Pulse

> 產出時間：2026-07-12 · 分支 `claude/codebase-trace-documentation-3ik4j3`（= master HEAD）
> 目的：為後續自動化技術文件產生任務提供素材，本文件**只讀不寫程式碼**。

## 專案總覽

Mini Taiwan Pulse 是一個 React 19 + TypeScript + Vite 的地理資訊視覺化前端專案，用 Mapbox GL + Three.js（搭配部分 deck.gl）在單一地圖上呈現台灣「天空（航班）、海洋（船舶）、大地（軌道列車）、街道（公車）」四種即時交通脈動，並疊加超過 30 種基礎設施 / 環境 / 災害 / 農漁業等靜態與動態圖層；後端資料源以 Supabase（PostgreSQL + PostGIS，`realtime` / `reference` / `spatial` / `public` 四個 schema 分工，前端一律經 `public.*` RPC 或 `reference` / `spatial` 直讀，禁止直打 `realtime.*`）為主，輔以 S3 / R2 上的大型靜態 GeoJSON、H3 預聚合 JSON、PMTiles 向量切片，最終以 Docker + nginx 部署（多個 `location` 區塊區分「大檔走 S3 volume `/data`」與「小檔走 git-tracked `dist` fallback」）。專案採單人開發 GitHub Flow，並搭配一套相當成熟的 Claude Code 協作機制（`.claude/memory` 記憶迴圈、`.claude/pitfalls` 已知坑、`.claude/skills` 領域技能、slash commands）。

## 技術棧

| 類別 | 技術 | 版本 / 說明 |
|---|---|---|
| 語言 | TypeScript | ~5.7.0，`strict` + `noUncheckedIndexedAccess` + `noUnusedLocals/Parameters` 全開 |
| 框架 | React | ^19.0.0，`react-dom` ^19.0.0 |
| 建置工具 | Vite | ^6.1.0，port 3721（`strictPort`），custom plugin 於 build 後從 dist 移除大型腳本輸入檔（如 `rail_bundle.json`） |
| TS 專案結構 | tsconfig project references | `tsconfig.json`（root，僅 references）→ `tsconfig.app.json`（`include: ["src"]`），`build` script 用 `tsc -b`（**禁用 `--noEmit`**，見 CLAUDE.md） |
| 地圖引擎 | Mapbox GL JS | ^3.9.0；另有 `mapbox-pmtiles` ^1.0.56 + `pmtiles` ^4.4.1 做向量切片圖層 |
| 3D 渲染 | Three.js | ^0.172.0，`src/three/*Scene.ts` 為各圖層專屬 scene（光球、拖尾、燈塔光束、溫度波浪曲面等），透過 CustomLayer 接入 Mapbox |
| 輔助渲染 | deck.gl | `@deck.gl/core` / `geo-layers` / `layers` / `mapbox` ^9.2.10（部分圖層用） |
| 後端 | Supabase | `@supabase/supabase-js` ^2.101.1；PostgreSQL + PostGIS，schema 分工見下 |
| 空間運算 | h3-js | ^4.4.0，六角格人口 / 人流密度圖層 |
| 衛星軌道計算 | satellite.js | ^5.0.0 |
| LLM / Chat | Vercel AI SDK | `ai` ^7.0.11 + `@ai-sdk/anthropic` / `google` / `openai`（BYOK 多家 LLM 聊天功能，見 `src/chat/`） |
| 雲端儲存 | AWS SDK S3 | `@aws-sdk/client-s3` ^3.995.0（`scripts/deploy/*` 上傳大檔到 S3） |
| 測試 | Vitest | ^3.2.6，`vitest.config.ts` 限定 `src/**/*.test.ts`，`environment: "node"` |
| 圖示 | lucide-react | ^0.576.0 |
| Schema 驗證 | zod | ^4.4.3 |
| CI | GitHub Actions | `.github/workflows/ci.yml`：`npm ci` → `npm run build`（`tsc -b` + `vite build`）→ `npm test`（vitest）；另有 `claude-mention.yml`、`claude-review.yml` |
| 部署 | Docker + nginx + Zeabur | 兩階段 Dockerfile（`node:22-alpine` build → `nginx:alpine` serve），`entrypoint.sh` 啟動前 pull S3 assets 到 `/data`；`docker-compose.yml` 掛載大型靜態檔到 `/data`；nginx 依路徑（`/geo/`、`/h3/`、`/bus/`、`/forestry/`、`/fishery/`、`/agriculture/`、`/medical/`、`/base_map/`、`/climate/`、`/fire/`、`/police_justice/`、`/road/`、`/rail/`、`/static-rpc/`、`/coverage/`）分流「S3 大檔優先 + dist fallback」；CSP 目前為 Report-Only 階段（BC-4 安全加固） |
| 資料收集 Python | tsx（scripts） + Python 3 | `scripts/fetch/*.ts`（TDX/FlightRadar24 等 API）、`scripts/preprocess/*.py`（`bundle-rail-data.py`、`generate-station-pillars.py`）、`scripts/export`、`scripts/deploy`、`scripts/audit`、`scripts/poc` |

## 目錄結構（annotated）

```
mini-taiwan-pulse/
├── src/                          # 373 個原始檔（.ts + .tsx，含 test）
│   ├── data/            (75)     # Supabase RPC / 靜態檔 loader，每個資料源一支 *Loader.ts，皆需包 loadingRegistry
│   │   └── __tests__/            # loader 單元測試
│   ├── hooks/            (91)    # React hook 層，use*Layer.ts 對應每個地圖圖層，橋接 loader → CustomLayer/overlay
│   │   └── factories/            # hook 工廠（重複 pattern 抽象）
│   ├── map/              (38)    # Mapbox CustomLayer 實作 + overlayRegistry（靜態圖層註冊）+ overlayManager/overlayPaintDiff（動態 paint diff）+ 色階（climateRamps/aqiColorScale）
│   │   ├── MapView.tsx            # 地圖主容器
│   │   └── __tests__/
│   ├── three/            (~20)   # Three.js Scene 類別，一個資料類別一個 Scene（Flight/Ship/Rail/Bus/Lighthouse/PowerXxx/Waste* 等），供 CustomLayer 使用；shaders/ 子目錄放 GLSL
│   ├── state/                    # 外部 store（非 React state）：timeStore.ts（timeline 訂閱，禁止塞進 useEffect deps）、chatStore、dateNotifier、realEstatePointsStore、satelliteConsoleStore
│   ├── types/                    # index.ts 為集中型別定義（含 LayerVisibility interface，新增 layer 必改第一步）
│   ├── lib/                      # 橫切工具：loadingRegistry（loading task 註冊中心）、supabase.ts（client）、auth.ts、keyVault.ts（BYOK 金鑰）、layerGates.ts（owner-gated layer）、rpcDebounce、dayPrefetch、sessionTracker
│   ├── components/               # UI 元件：admin/、auth/、chat/、featureInfo/（click popup registry）、hud/、intel/、satelliteConsole/、sidebar/（layerCatalog.ts = LAYER_COLORS + SECTIONS 單一真實來源）
│   ├── chat/                      # BYOK 多 LLM 聊天：agent.ts、providers.ts、systemPrompt.ts、tools/
│   ├── engines/                   # BusEngine.ts / RailEngine.ts / TraTrainEngine.ts（progress-based 動畫引擎，見 bus-layer-design.md）
│   ├── constants/、utils/         # 型別常數、座標轉換、插值、maneuver impact 等純函式
│   └── styles/                    # CSS
├── scripts/
│   ├── fetch/            # 外部 API 抓取（TDX、FlightRadar24 等），對應 package.json `fetch:*` scripts
│   ├── preprocess/       # Python 前處理（rail bundle、station pillars）
│   ├── deploy/           # S3 上傳 + nginx entrypoint/pull-deploy-assets/refresh-climate
│   ├── export/           # DB 匯出
│   ├── audit/            # 稽核腳本
│   └── poc/              # POC / 實驗腳本
├── public/               # 靜態資產（扁平檔名契約，不可改路徑），子目錄對應 nginx location：base_map/ bus/ climate/ coverage/ fishery/ flood/ forestry/ geo/ h3/ + station_pillars.json + three-showcase.html（68 元件 3D 案例庫）
├── docs/                 # 185 個 md 檔（含子目錄）
│   ├── development-rules.md, TIMELINE_ARCHITECTURE.md, supabase-optimization.md,
│   │   known-issues.md, bus-layer-design.md         # 核心規則/架構文件（見下方索引）
│   ├── features/         # 19 個 feature 資料夾（agriculture/aquaculture/bloom-experiments/bus/byok-chat/
│   │                       er-hospital/fire-rescue/global-climate/imagery/livestock/news/
│   │                       owner-gated-layers/parking/real-estate/road-congestion/static-to-cdn/
│   │                       terrain-vector/tourist-shuttle/water-resources + _TEMPLATE）
│   │                       每個資料夾慣例含 README/backlog/changelog/handoff
│   ├── audit/, design/, images/, launch/, proposal/, research/   # 其他文件分類
│   └── 其餘散落頂層 md：NEWS_MAP_PLAN, codex-workflow, design-system, energy-*-status/plan,
│         intel-panel-status, medical-layers-plan, perf-*, scaling-resilience-runbook,
│         session-analytics, supabase_rpc_audit, three-showcase-library, waste-collection-status,
│         water-opendata-catalog, water-resources-status
└── .claude/
    ├── memory/           # STATUS/BACKLOG/PRINCIPLES/DATA_SCOPE/GLOSSARY/INCIDENTS/PLAYBOOKS/
    │                       PMTILES_STATUS/FORESTRY_GROUP_STATUS/REFLECTIONS.md + load-session.sh
    ├── pitfalls/         # 5 個具名 bug 復盤（如 2026-04-07-empty-ships-flights.md）+ _TEMPLATE
    ├── skills/           # accessibility-analysis, layer-onboarding, service-coverage,
    │                       supabase-optimize, three-3d-component, wrap-up
    ├── commands/         # check-rpc.md, handoff.md, new-layer.md
    ├── agents/           # layer-creator.md
    └── FRAMEWORK.md, HARNESS.md, README.md, SATELLITE_CONSOLE_STATUS.md
```

**架構模式判定**：這是一個**單頁應用（monolith SPA）+ Plugin-based Layer 系統**的混合體 —— 不是微前端，而是一個 React 主殼（`App.tsx` + `MapView.tsx`）搭配高度規格化的「圖層插件」慣例：每新增一個資料圖層都遵循固定 7 步流程（type → loader → hook → CustomLayer/overlayRegistry → layerCatalog 註冊 → App.tsx 接線 → 預設可見性），並強制搭配 4 條 UX 鐵則（opacity slider、圖例、click popup、`<select>` dropdown）。這種「約定優於配置」的重複 pattern 使 91 個 hook / 75 個 loader / 38 個 CustomLayer 檔案能保持一致的可維護性，也解釋了為何專案投入大量心力在 slash command（`/new-layer`）與 skill（`layer-onboarding`）自動化骨架產生與驗收上。時間軸則用「外部 store（`timeStore.ts`）+ 訂閱制」取代 React state，避免高頻 timeline 更新觸發整棵樹 re-render——這是效能導向的架構決策，明確寫入 CLAUDE.md 強制規則第 6 條。

## 既有文件索引

| 路徑 | 一句話摘要 | 可否直接引用 |
|---|---|---|
| `CLAUDE.md` | 精簡版開發規則索引：資料來源分工、loading UI 規範、pre-aggregate pattern、新增 layer 7 步 + UX 四鐵則、動態圖層時間訂閱、Git workflow | 可（trace 文件應直接連結而非重述） |
| `docs/development-rules.md`（341 行） | 開發規則詳細版，含完整 rationale + 程式碼範例（loading pattern、schema 分工表、pre-aggregate SOP、7 步流程檢查清單、UX 四鐵則細節） | 可，是本次 trace 文件「資料流 / 慣例」章節的權威來源 |
| `docs/TIMELINE_ARCHITECTURE.md`（338 行） | 多源彈性時間軸設計：三層 UI 結構（底部時間軸 / 模式切換 / 逐層控制）、資料密度條、非線性時間壓縮構想 | 可，timeline 相關章節直接引用 |
| `docs/supabase-optimization.md`（139 行） | Pre-aggregate pattern 完整指南：4 組件（table/refresh function/cleanup/pg_cron）、SQL 範本位置、Supavisor pooler 2 分鐘 timeout 限制 | 可，資料庫優化章節引用 |
| `docs/known-issues.md`（78 行） | 歷史事故復盤：Zeabur 重啟斷層、時區 bug（`datetime.now()` naive → +8h 偏移）、2026-04-09 IO/CPU 爆表事件（cron 未錯開 + 遺忘的舊 MV refresh job）+ 診斷指令 | 可，運維 / 已知問題章節可直接連結，不需重寫 |
| `docs/bus-layer-design.md`（390 行） | 公車圖層 progress-based 時間軸架構：為何不能直接用 GPS 座標、trip segmentation、GPS 異常處理，全台擴展參考 | 可，作為「複雜圖層」案例研究直接引用 |
| `docs/features/*/`（19 個資料夾） | 各 feature 的 README/backlog/changelog/handoff（agriculture、aquaculture、bus、byok-chat、er-hospital、fire-rescue、global-climate、imagery、livestock、news、owner-gated-layers、parking、real-estate、road-congestion、static-to-cdn、terrain-vector、tourist-shuttle、water-resources、bloom-experiments） | 可，逐 feature 的細節不需在 trace 文件重複，用連結索引即可；`docs/features/README.md` 本身即為索引頁，值得直接參考其結構 |
| `docs/supabase_rpc_audit.md` | RPC 效能盤點清單，追蹤哪些 RPC 已套 pre-aggregate | 可，資料庫章節引用 |
| `.claude/memory/STATUS.md` / `BACKLOG.md` / `PRINCIPLES.md` | Session 記憶迴圈核心：目前進度、待辦、P0 原則 | 建議引用但不摘要內容（會隨時間變動，trace 文件應連結而非複製） |
| `.claude/pitfalls/*.md`（5 篇） | 具名 bug 復盤模板化紀錄 | 可作為「常見陷阱」章節案例 |
| `docs/design-system.md`、`docs/three-showcase-library.md` | 視覺設計系統 / Three.js 68 元件案例庫說明 | 可，UI/3D 章節引用 |
| `docs/perf-*.md`（3 篇：`perf-external-time-store.md`、`perf-optimization-2026-04-14.md`、`perf-overhaul-2026-06.md`）、`docs/perf-p0a-test-plan.md` | 效能優化歷程紀錄 | 可作為效能章節時間軸引用 |
| `docs/scaling-resilience-runbook.md` | 擴展性 / 容錯 runbook | 可，維運章節引用 |
| `docs/NEWS_MAP_PLAN.md`、`docs/medical-layers-plan.md`、`docs/energy-*-plan.md` | 個別 feature 規劃文件（部分已落地、部分仍是 plan） | 部分可引用，需先核對現況是否已實作（避免文件與程式碼不一致） |

**結論**：既有文件覆蓋度非常高（151+ 個 md 檔，且已有嚴謹的 `/wrap-up` 更新機制維持新鮮度），Stage 2+ 的 trace 文件產出應**優先建立索引與交叉連結**，避免重複造輪子；真正有價值補強的是「把分散在 91 個 hook / 75 個 loader / 38 個 CustomLayer 中的**重複模式抽象成一張總覽圖**」，以及「新人 onboarding 視角的端到端資料流講解（Supabase RPC → Loader → Hook → CustomLayer → Three.js Scene → Mapbox render）」，這是目前文件比較分散、沒有單一篇幅講完整條鏈路的部分。

## 檔案規模統計

- **Git tracked 總檔案數**：738
- **依副檔名分類**（前 10 大）：
  | 副檔名 | 數量 | 說明 |
  |---|---|---|
  | `.ts` | 293 | TypeScript 原始碼（含 scripts） |
  | `.md` | 185 | 文件 |
  | `.tsx` | 86 | React 元件 |
  | `.geojson` | 38 | 靜態地理資料（git-tracked 小檔） |
  | `.json` | 37 | 設定檔 + 小型資料檔 |
  | `.py` | 33 | Python 前處理腳本 |
  | `.png` | 14 | 截圖 / 圖片 |
  | `.pmtiles` | 14 | 向量切片（git-tracked 小檔，大檔在 S3） |
  | `.js` | 11 | 少量 JS |
  | `.sh` | 8 | Shell 腳本（deploy/entrypoint） |
- **`src/` 純程式碼檔案**（`.ts` + `.tsx`，排除 `.test.ts`）：**353 個**
- **`src/` 測試檔案**（`.test.ts`）：**17 個**（vitest，`vitest.config.ts` 限定 `src/**/*.test.ts`）
- **`docs/` 總計**：151+ 個 md 檔（依任務描述），實查頂層 + `features/` 子目錄結構符合

## 初步觀察到的架構模式

1. **Layer-plugin 慣例架構**：不是抽象出通用 plugin 介面，而是「命名慣例 + 強制檢查清單」達成一致性（7 步流程 + TS 編譯期 `Record<keyof LayerVisibility, string>` 強制 `LAYER_COLORS` 補齊，漏改會直接 `tsc` 報錯 TS2739）。這是一種「用型別系統做架構約束」的實用做法，而非傳統的抽象基底類別。
2. **資料源三層分工明確**：Supabase RPC（動態時序）/ 靜態 GeoJSON（`public/`，扁平檔名契約）/ 大型預聚合檔（S3 + nginx location 分流），且有清楚的「何時該套 pre-aggregate pattern」判斷標準（>1s 或 >10k rows）。
3. **時間狀態外部化**：`timeStore.ts` 用 subscribe 模式而非 React state/context 處理 timeline，避免高頻更新造成 re-render 風暴 —— 這是專案級強制規則（CLAUDE.md 第 6 條），也是與一般 React SPA 架構的顯著差異點。
4. **Three.js × Mapbox CustomLayer 混合渲染**：`src/three/*Scene.ts`（場景邏輯）與 `src/map/*CustomLayer.ts`（Mapbox CustomLayer 生命週期橋接）分離，讓每種視覺語意（光球、拖尾、光柱、光束）可重用共用元件（`LightOrb.ts`、`LightTrail.ts`、`GlowPointsScene.ts`），並有專屬 skill（`three-3d-component`）與 68 元件案例庫（`public/three-showcase.html`）支援新增 3D 元件的決策。
5. **BYOK 多 LLM 聊天子系統**：`src/chat/` + `@ai-sdk/*` 是相對獨立的子系統（agent/providers/systemPrompt/tools），透過 CSP `connect-src` 白名單限制金鑰只能送往三家 LLM + Supabase + Mapbox，顯示專案已進入「面向公開使用者」的安全加固階段（`nginx.conf` 中的 BC-4 安全 header 注解也印證此點：2026-07-04 新增，CSP 目前 Report-Only）。
6. **強 Claude Code 協作基礎設施**：`.claude/` 下有完整的 memory 迴圈、pitfalls 知識庫、多個領域 skill、slash command 自動化（`/new-layer`、`/check-rpc`、`/wrap-up`），顯示這是一個長期由 AI agent 高度參與維護的專案，文件與程式碼的同步機制比一般專案更嚴謹（`/wrap-up` 明確定義何時更新哪個 memory 檔）。
7. **多 repo 協作生態**：`mini-taiwan-pulse` 為前端，`gis-platform`（Supabase migrations）、`data-collectors`（資料收集 + SQL 範本）、`pulse-api`（FastAPI 備援，目前已不接前端）、`mini-taipei-v3`（鐵道資料源）為姊妹 repo，CLAUDE.md 明確定義「上游先動、下游後動」的跨 repo 同步順序，Stage 2+ 若要完整記錄資料流，需注意部分邏輯（refresh function、collector）並不在本 repo 內。

---

## Stage 1.5 範圍評估（Main Agent 附註）

- `git ls-files` 總數 738 個，形式上超過 500 檔門檻，但扣除既有文件（185 個 `.md`）、靜態資料（38 個 `.geojson`）、圖片/PMTiles 資產（28 個）後，實際純程式碼檔案（`.ts`/`.tsx`，不含測試）為 353 個，低於 500。
- 判斷：**不縮限範圍，進行 full trace**。理由：(1) 核心程式碼規模適中；(2) 既有文件雖多但分散在 19 個 feature 資料夾，缺乏「端到端資料流」與「架構總覽」單一入口，這正是本次 trace 最大價值所在；(3) 私有內部專案沒有外部貢獻者急迫性，值得一次做完整。
- 專案類型判定：**Frontend SPA**，但因外部整合（Supabase RPC / TDX API）是架構核心且複雜度高，**不採用「Frontend SPA 可跳過 2.5」的預設**，2.5 外部整合維持獨立文件。
- 2.2 Request/Data Flow 因無傳統 request/response server，改為 trace「圖層資料生命週期」：Supabase RPC → Loader → Hook → CustomLayer/Scene → Mapbox render。
