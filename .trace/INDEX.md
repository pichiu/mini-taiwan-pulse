# INDEX — Mini Taiwan Pulse 專案總覽與速查

> 本文件為 `.trace/` 文件集的入口。所有文件由 Claude Code 對 codebase 進行自動化 trace 產出，基準 commit 見 [`TRACE_META.md`](./TRACE_META.md)。

## 一句話總覽

Mini Taiwan Pulse 是一個 React 19 + TypeScript + Vite 前端專案，用 Mapbox GL JS + Three.js 在單一互動地圖上呈現台灣天空（航班）、海洋（船舶）、大地（軌道）、街道（公車）等即時交通脈動，並疊加超過 30 種基礎設施 / 環境 / 災害 / 農漁業圖層；資料源以 Supabase（PostgreSQL + PostGIS）為主，服務對象是需要即時掌握台灣各類公共資料的一般使用者與研究者。

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|---|---|---|---|
| 語言 | TypeScript | ~5.7 | `strict` + `noUncheckedIndexedAccess` 全開 |
| 框架 | React | ^19.0.0 | 單一巨型 SPA（無路由），`App.tsx` 2870 行為主容器 |
| 建置 | Vite | ^6.1.0 | port 3721（`strictPort`），自訂 plugin 移除 build 產物中的中繼大檔 |
| 地圖引擎 | Mapbox GL JS | ^3.9.0 | 底圖 + vector tile + CustomLayer 掛載點 |
| PMTiles | mapbox-pmtiles / pmtiles | ^1.0.56 / ^4.4.1 | 向量切片離線圖層 |
| 3D 渲染 | Three.js | ^0.172.0 | 掛在 Mapbox CustomLayer 上的粒子/光柱/波浪等視覺 |
| 輔助渲染 | deck.gl | ^9.2.10 | 部分圖層使用 |
| 後端 BaaS | Supabase (`@supabase/supabase-js`) | ^2.101.1 | PostgreSQL + PostGIS，`realtime`/`reference`/`spatial`/`public` 四 schema 分工 |
| 空間索引 | h3-js | ^4.4.0 | 六角格密度圖層 |
| LLM | Vercel AI SDK (`ai` + `@ai-sdk/*`) | ^7.0.11 | BYOK 多 LLM 聊天（Anthropic/OpenAI/Google） |
| 測試 | Vitest | ^3.2.6 | `src/**/*.test.ts`，含 `layerConsistency` ratchet 測試 |
| CI | GitHub Actions | — | `npm ci && npm run build && npm test` |
| 部署 | Docker + nginx + Zeabur | — | 兩階段建置，S3/R2 大檔 + git-tracked dist 雙軌靜態資產 |

## 關鍵指令速查

```bash
pnpm dev          # 啟動 dev server（port 3721）
npx tsc -b        # TypeScript 驗證（commit 前必跑，project references，禁用 --noEmit）
pnpm test         # vitest run（含 layerConsistency 圖例/擴充一致性守備測試）
pnpm build        # tsc -b && vite build
```

## 文件地圖

| 文件 | 內容 | 適合誰 |
|---|---|---|
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | 系統架構、元件清單、通訊模式、關鍵設計決策 | 想理解整體架構的新人 |
| [`DATA_MODEL.md`](./DATA_MODEL.md) | 前端消費的資料形狀、Supabase RPC 對照、狀態管理策略 | 要接新資料源的開發者 |
| [`API_SURFACE.md`](./API_SURFACE.md) | Supabase RPC 清單、外部整合介面、CLI/npm scripts | 要新增 loader 或除錯資料層的開發者 |
| [`DEV_GUIDE.md`](./DEV_GUIDE.md) | 環境建置、開發 workflow、測試、常見踩坑 | 第一次 clone 這個 repo 的人 |
| [`CODEBASE_MAP.md`](./CODEBASE_MAP.md) | 目錄地圖、「我想改 X 要看哪裡」速查表 | 要動手改程式碼前 |
| [`DISCOVERY_LOG.md`](./DISCOVERY_LOG.md) | 文件與程式碼落差、TODO 彙整、待解問題 | 想知道哪裡還不確定 / 有技術債 |

## 專案專屬術語表

| 術語 | 意思 |
|---|---|
| Layer-plugin 架構 | 本專案「新增圖層」的慣例：type → loader → hook → CustomLayer/overlayRegistry → layerCatalog → App.tsx → DEFAULT_ON 七步流程，靠 TypeScript 型別窮舻（`Record<keyof LayerVisibility, string>`）與 ratchet 測試強制一致性，而非傳統 plugin 基底類別 |
| `LayerVisibility` | `src/types/index.ts` 中定義的單一真實來源 interface，每個圖層對應一個 boolean key |
| `overlayRegistry` | `src/map/overlayRegistry.ts`，泛型化登記靜態 GeoJSON/PMTiles 圖層的巨型設定檔（6400+ 行），由 `MapView.tsx` 統一查表決定顯隱 |
| `timeStore` | `src/state/timeStore.ts`，外部 Observer/Pub-Sub store，取代 React state 處理 timeline，避免高頻 tick 造成整棵樹 re-render；四種訂閱粒度：`subscribe`（逐幀）/`subscribeThrottled(ms)`/`subscribeDate`（debounce 換日）/`getTime()`（同步讀） |
| `wallClock` | 與 `timeStore` 平行、但代表「真實時間」（非資料時間軸）的獨立 tick 池，給「幾分鐘前」類 UI 用，容易與 `timeStore` 混淆 |
| `loadingRegistry` | `src/lib/loadingRegistry.ts`，全域 loading task 註冊中心，所有 Supabase 非同步呼叫依專案規則必須包 `start()`/`complete()` |
| `resilientFetch` | `src/lib/supabase.ts` 注入 Supabase client 的自訂 fetch：併發上限 8、30s timeout、429/5xx retry |
| Pre-aggregate pattern | Supabase RPC 響應 >1s 或 >10k rows 時強制套用的優化模式：普通 table + per-day refresh function + pg_cron + 薄 SELECT RPC，繞開 Supavisor pooler 2 分鐘 timeout |
| `layerCatalog.ts` | `src/components/sidebar/layerCatalog.ts`，`LAYER_COLORS` + `SECTIONS` 的單一真實來源，同時供桌機 `IconRailSidebar` 與手機 `LayerSidebar` 兩個 UI 消費 |
| UX 四鐵則 | 任何新 layer 必過：透明度 slider / (分類≥2)必寫圖例 / 可選取物件必接 click popup / (options≥4)用原生 `<select>` |
| ratchet 測試 | `layerConsistency.test.ts` 等測試用「已知例外白名單」擋新增遺漏，同時允許登記過的舊技術債存在 |
| 姊妹 repo | `gis-platform`（Supabase migrations）/ `data-collectors`（資料收集 + SQL 範本）/ `pulse-api`（FastAPI 備援，已停用）/ `mini-taipei-v3`（鐵道資料源），本 repo 前端不含它們的邏輯 |

## 最有趣的技術決策速覽

見 Stage 4 最終摘要（本次對話結尾）與 [`ARCHITECTURE.md`](./ARCHITECTURE.md) 的「關鍵設計決策」章節。
