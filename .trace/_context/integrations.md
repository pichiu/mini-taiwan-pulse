# 外部整合追蹤 — Mini Taiwan Pulse

> Stage 2 產出 · 只讀程式碼分析，未修改任何檔案
> 對應背景：`.trace/_context/recon.md`（Stage 1 偵察）、`.trace/_context/web_findings.md`（Stage 1.5 線上資源）

本文件盤點前端（`mini-taiwan-pulse` repo）實際串接的所有外部系統：Supabase（主要後端）、Mapbox GL + PMTiles（地圖渲染）、BYOK 多 LLM（Anthropic/OpenAI/Google，瀏覽器直連）、Cloudflare R2 CDN（影像快取）、以及部署層（Docker + nginx + S3 + Zeabur + Cloudflare）。**TDX 等原始資料收集 API 不在本 repo**，僅在姊妹 repo `data-collectors` 處理，本 repo 前端純消費 Supabase / CDN 產出物。

---

## 總覽圖

```mermaid
graph TB
    subgraph Browser["瀏覽器（React 19 SPA）"]
        App[App.tsx]
        Loaders["src/data/*Loader.ts\n(75 個)"]
        Hooks["src/hooks/use*Layer.ts\n(91 個)"]
        Map["src/map/*CustomLayer.ts\n(38 個) + MapView.tsx"]
        Chat["src/chat/\nBYOK 多 LLM"]
        KeyVault["src/lib/keyVault.ts\n(BYOK key 存 localStorage)"]
    end

    subgraph Supabase["Supabase（gis-platform）"]
        RPC["public.* RPC\n(get_xxx_current / get_xxx_day ...)"]
        REST["PostgREST /rest/v1/\n(satellite_classified view\nlog_session_events)"]
        Realtime[["realtime schema\n（前端禁止直打）"]]
        Reference[("reference / spatial schema")]
    end

    subgraph CDN["靜態資產 CDN"]
        S3["S3（nginx entrypoint pull → /data）"]
        R2["Cloudflare R2\ndata.itsmigu.com\n(CWA 衛星/雷達影像)"]
        Dist["git-tracked dist fallback\n(public/ 扁平檔名)"]
        PMTiles["public/*.pmtiles (14 檔)\n+ /data/*/**.pmtiles"]
    end

    subgraph LLM["BYOK 第三方 LLM（瀏覽器直連，無 proxy）"]
        Anthropic[api.anthropic.com]
        OpenAI[api.openai.com]
        Google[generativelanguage.googleapis.com]
    end

    subgraph MapboxCloud["Mapbox"]
        MapboxAPI[api.mapbox.com]
        MapboxTiles["*.tiles.mapbox.com"]
        MapboxEvents[events.mapbox.com]
    end

    subgraph Edge["部署 / Edge"]
        Zeabur[Zeabur 容器]
        Cloudflare[Cloudflare CDN/WAF]
    end

    Loaders -->|supabase-js .rpc()\n+ resilientFetch| RPC
    Loaders -->|raw fetch + apikey header| REST
    RPC --> Reference
    REST --> Reference
    Realtime -. "禁止直打，僅 public wrapper 讀" .-> RPC

    Loaders -->|staticRpc() 靜態化快照優先\n404 fallback 回 RPC| S3
    Loaders -->|VITE_IMAGERY_CDN_BASE| R2
    Map -->|pmtiles:// protocol| PMTiles
    S3 -.->|entrypoint.sh 啟動時 pull| Zeabur
    Dist -.->|try_files fallback| Zeabur

    Chat -->|apiKey from KeyVault\nheaders: anthropic-dangerous-direct-browser-access| Anthropic
    Chat --> OpenAI
    Chat --> Google
    KeyVault -.-> Chat

    Map -->|accessToken| MapboxAPI
    Map --> MapboxTiles
    Map --> MapboxEvents

    Zeabur --> Cloudflare
    Cloudflare --> Browser

    style Realtime fill:#f66,color:#fff
```

---

## 1. Supabase 整合

### 1.1 Client 初始化

`src/lib/supabase.ts:1-153`

- SDK：`@supabase/supabase-js` `^2.101.1`（`createClient`，`src/lib/supabase.ts:1`）
- Env 讀取（`src/lib/supabase.ts:3-4`）：
  ```ts
  const SUPABASE_URL = import.meta.env.VITE_SUPABASE_URL ?? import.meta.env.SUPABASE_URL;
  const SUPABASE_ANON_KEY = import.meta.env.VITE_SUPABASE_ANON_KEY ?? import.meta.env.SUPABASE_ANON_KEY;
  ```
  雙重 fallback（`VITE_*` 優先，非 `VITE_*` 前綴備援），Vite 只會 inline `VITE_*` 前綴變數，故非 `VITE_` 版本實務上多半是空值，屬防禦性寫法。
- **缺 env 不會 crash app**：`createStubClient()`（`src/lib/supabase.ts:19-38`）用 `Proxy` 包出一個永遠 `resolve({ data: null, error })` 的假 client，`supabaseConfigured` 布林旗標（`src/lib/supabase.ts:6`）供上層判斷是否真的連得上。
- **AR-01 韌性 fetch wrapper**（`src/lib/supabase.ts:40-141`，透過 `global: { fetch: resilientFetch }` 注入 supabase-js，`src/lib/supabase.ts:143-147`）：
  - 全域併發上限 8（`MAX_CONCURRENT_REQUESTS`，`src/lib/supabase.ts:51`），超過進 FIFO `waitQueue`
  - 每請求 30s timeout（`REQUEST_TIMEOUT_MS`，`src/lib/supabase.ts:52`），`AbortSignal.any()` 合成 caller 自帶 signal
  - 網路層 `TypeError` 或 5xx/429 最多 retry 2 次，backoff 500ms/1500ms + jitter（`src/lib/supabase.ts:53-54, 91-94`）
  - 寫入類 RPC denylist 不 retry（`WRITE_RPC_DENYLIST = ["log_session_events"]`，`src/lib/supabase.ts:57`），避免重送造成重複寫入
  - 錯誤不吞，最終仍 throw／轉成 `{ error }`（`src/lib/supabase.ts:48-49` 註解明講「loadingRegistry 錯誤態依賴這點」）

### 1.2 RPC 命名慣例（代表性樣本，實際呼叫點 121 處、跨 75 個 loader 檔）

用 `grep -rn 'supabase\.rpc(' src/data` 統計得出。命名模式可歸納三類：

| 模式 | 語意 | 範例 |
|---|---|---|
| `get_xxx_current` | 當下即時快照（LIVE 圖層） | `get_bus_current`（`src/data/busLoader.ts:73`）、`get_parking_segments_current`（`src/data/parkingLoader.ts:136`）、`get_bus_intercity_current`（`src/data/busLoader.ts:225`） |
| `get_xxx_latest` | 最新一筆（通常對應 timeline 尚未接入的簡單層） | `get_er_hospital_latest`（`src/data/erHospitalLoader.ts:83`）、`get_uswg_latest`（`src/data/floodSensorLoader.ts:41`）、`get_aqi_stations_latest`（`src/data/aqiStationsLoader.ts:73`） |
| `get_xxx_day` / `_dates` | Timeline scrub：`_dates` 先列有資料的日期、`_day` 拉單日資料 | `get_freeway_dates` + `get_freeway_congestion_day`（`src/data/freewayLoader.ts:43,88`）、`get_road_events_dates`/`_day`（`src/data/roadEventsLoader.ts:71,89`）、`get_parking_dates`（`src/data/parkingLoader.ts:371`） |
| `get_xxx_timeseries` | 單一測站/物件的長條時序（折線圖用） | `get_river_water_level_timeseries`（`src/data/riverLevelLoader.ts:60`）、`get_groundwater_timeseries`（`src/data/groundwaterLoader.ts:89`）、`get_reservoir_timeseries`（`src/data/reservoirOpsLoader.ts:65`） |
| `get_xxx_trails` | 移動軌跡（公車/船舶/飛機拖尾） | `get_bus_trails`（`src/data/busLoader.ts:133`）、`get_ship_trails`（`src/data/shipLoader.ts:85`）、`get_flight_trails`（`src/data/airspaceLoader.ts:79`） |
| `get_ssot_*` | Single Source of Truth 電廠/設施資料（多來源整併） | `get_ssot_power_plants_with_output`、`get_ssot_facilities_offshore_zones`、`get_ssot_facilities_provenance`（`src/data/energyLoader.ts:94,253,358`） |

其餘代表性樣本（共 121 呼叫點，節錄約 20 個以呈現命名廣度）：`get_livestock_farms`（`src/data/livestockLoader.ts:23`）、`get_disaster_alert_dates`/`_alerts_day`（`src/data/disasterAlertLoader.ts:73,91`）、`get_lightning_recent`/`_window`/`_day`（`src/data/lightningLoader.ts:28,55,84`）、`get_youbike_h3_dates`/`_snapshots`（`src/data/youbikeH3Loader.ts:41,55`）、`get_nuclear_radiation_status`/`_at`/`_day`（`src/data/nuclearLoader.ts:28,44,85`）、`get_h3_demographics_yearly`/`_years`（`src/data/h3Loader.ts:256,280`）、`get_news_events_day_clustered_v2`（`src/data/newsEventsLoader.ts:174,215`）、`get_taipei_sewer_latest`/`_evacuate_latest`/`_pumb_latest`（`src/data/wicTaipeiLoader.ts:35,68,100`）、`get_source_health`/`get_pressure_index_now`/`get_market_index_now`/`get_pla_activity_latest`（`src/data/intelLoaders.ts:51,139,220,283` — 情報儀表板類 RPC）、`get_fossil_fuel_infrastructure`/`get_fossil_fuel_layers`（`src/data/energyLoader.ts:743`、`src/data/fossilFuelLoader.ts:204`）、`get_cwa_imagery_list`/`_frame`（`src/data/cwaImageryLoader.ts:170,214`）、`get_data_catalog_for_layer`/`_by_theme`（`src/data/dataCatalogLoader.ts:88,119`）。

**命名慣例小結**：`get_` 前綴一律唯讀（符合 CLAUDE.md「一律 public RPC wrapper」原則），動詞後接資料主題（`bus`/`freeway`/`nuclear_radiation`…），再接時間粒度後綴（`_current`/`_latest`/`_day`/`_dates`/`_timeseries`/`_trails`）。全 repo 唯一非 `get_` 開頭、會寫入的 RPC 是 `log_session_events`（`src/lib/sessionTracker.ts:68`，session 分析事件緩衝上傳），在 `src/lib/supabase.ts:57` 的 `WRITE_RPC_DENYLIST` 中特別排除 retry。

### 1.3 RLS / anon key 安全模式如何在程式碼體現

- **前端只用 anon key**，且只呼叫 `public.*` schema 的 RPC 或 PostgREST view（`satellite_classified`，見 §1.4），未見任何 `schema('realtime')` 或直接 `.from('realtime.xxx')` 呼叫（`grep -rn "realtime\." src/data` 無匹配，符合 CLAUDE.md 禁令）。
- **RLS 執行邏輯本身不在本 repo**（在 `gis-platform` 的 migration），前端能看到的只有「呼叫哪個 RPC 名稱」這一層，安全邊界透過 Postgres function 的 `SECURITY DEFINER` + RLS policy 落地在資料庫端。
- 寫入操作被壓縮到單一 RPC（`log_session_events`），且其呼叫路徑（`src/lib/sessionTracker.ts:43-61` 的 `flushViaBeacon` 用 raw `fetch` + anon key header；`src/lib/sessionTracker.ts:63-` 的 `flushViaRpc` 用 `supabase.rpc`）都是「送分析事件」性質，非業務寫入，風險面被刻意收斂到最小。
- `src/data/staticRpc.ts:1-33` 的 `staticRpc()` 是一個「先打靜態 CDN JSON、失敗才 fallback 回真 RPC」的包裝層（見 §2/§5 static-to-cdn 說明），回傳形狀刻意對齊 `supabase.rpc` 的 `{ data, error }`，讓呼叫端（loader）不需改寫。

### 1.4 例外：不經 supabase-js 的直連 PostgREST 呼叫

`src/data/satelliteLoader.ts:14-62` 為效能/快取考量，**不用 supabase-js**，改用原生 `fetch` 直打 PostgREST view：
```ts
const url = `${SUPABASE_URL}/rest/v1/satellite_classified?select=norad_id,name,category,country_operator,tle_line1,tle_line2&${qs}`;
const resp = await fetch(url, { headers: { apikey: SUPABASE_ANON_KEY, Authorization: `Bearer ${SUPABASE_ANON_KEY}` } });
```
這條路徑**繞過了 `resilientFetch` 的併發上限/timeout/retry**（因為沒有走 `supabase.rpc`/`supabase.from`），僅靠 `localStorage` 6 小時快取（`src/data/satelliteLoader.ts:17-18,34-50`）降低請求頻率。程式碼註解說明選這條路徑是因為「CelesTrak `active.txt` 瀏覽器直連會被 403 擋（CORS/UA），改由 gis-platform 每 2h 從 Space-Track 同步進 `satellite_classified` view」（`src/data/satelliteLoader.ts:1-3`）。`src/lib/sessionTracker.ts:51` 的 `flushViaBeacon` 同樣是原生 `fetch`（`keepalive: true`，模擬 `sendBeacon` 但避開 JSON CORS 限制），也繞過 resilientFetch。

---

## 2. Mapbox GL 整合

### 2.1 Map 初始化

`src/map/MapView.tsx:191-204`：
```ts
mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;
const map = new mapboxgl.Map({
  container: containerRef.current,
  style: styleUrl,
  center: presetRef.current.center,
  zoom: presetRef.current.zoom,
  pitch: presetRef.current.pitch,
  bearing: presetRef.current.bearing,
  antialias: true,
});
```
- Token 型別宣告在 `src/vite-env.d.ts:4`（`readonly VITE_MAPBOX_TOKEN: string`），**build-time inline**（Dockerfile 用 `ARG`/`ENV` 在 build stage 注入，見 §5）。
- `styleUrl` 由外部傳入（未在本次追蹤範圍內展開來源，推測為 Mapbox style URL 或本地自訂 style JSON，屬 `cameraPresets.ts`/呼叫端決定）。
- `map.on("style.load", ...)`（`src/map/MapView.tsx:207-`）是唯一的底圖切換 handler：每次觸發都會依序 `applyPureBlackTheme` → `setupTerrain` → `registerPmtilesSourceTypeOnce()` → `resetOverlayHydration()` → `addAllOverlays(OVERLAY_REGISTRY, ...)`，確保切換底圖後所有 overlay 重建不遺漏。

### 2.2 PMTiles 整合

- 套件：`mapbox-pmtiles ^1.0.56` + `pmtiles ^4.4.1`（依 recon.md 技術棧表）
- **註冊點**：`src/map/pmtilesSourceType.ts:15-33` 的 `registerPmtilesSourceTypeOnce()`，在 `style.load` handler 內、`addAllOverlays` 之前呼叫（`src/map/MapView.tsx:212-213`）。核心邏輯：
  ```ts
  import { PmTilesSource } from "mapbox-pmtiles/dist/mapbox-pmtiles.js";
  const actualType = (PmTilesSource as unknown as { SOURCE_TYPE: string }).SOURCE_TYPE;
  Style.setSourceType(actualType, PmTilesSource);
  ```
  用模組級 `registered` flag 防重複註冊（`src/map/pmtilesSourceType.ts:13,16`），並主動比對套件升版後 `SOURCE_TYPE` 常數是否與本地 `PMTILES_SOURCE_TYPE`（`src/map/pmtilesConstants.ts`）一致，不一致會 `console.error` 而非靜默失效（`src/map/pmtilesSourceType.ts:19-24`）。檔頭註解說明這是**收斂重構**：原本 agriculture/fire/medical 三個 factory 各自維護一份重複邏輯，且彼此有隱性呼叫順序依賴，現統一到單一模組。
- **14 個 git-tracked PMTiles 檔案**（`find public -iname "*.pmtiles"`）分布：
  - `public/forestry/`：`forest_roads.pmtiles`、`trail_coverage.pmtiles`、`forest_reserve.pmtiles`、`signal_gap.pmtiles`、`national_forest_compartments.pmtiles`（5 檔）
  - `public/coverage/`：`taiwan_all_gas_nearest.pmtiles`、`taiwan_taisugar_nearest.pmtiles`、`taiwan_other_nearest.pmtiles`、`taiwan_cpc_nearest.pmtiles`、`aviation_airspace.pmtiles`、`taiwan_ev_nearest.pmtiles`、`drone_restricted_zones.pmtiles`、`taiwan_fpcc_nearest.pmtiles`（7 檔）
  - `public/flood/`：`uswg_isochrone_3min.pmtiles`（1 檔）
  這些是「git 小檔」版本（nginx `/coverage/`、`/forestry/` location 有 `try_files $uri @dist` fallback，見 §5）；更大的 PMTiles（`medical/`、`base_map/`、`climate/`、`fire/`、`police_justice/`、`road/`、`water_resources/`、`agriculture/`、`fishery/`）純走 S3、不進 git（nginx.conf 各 location 註解明確標註「純 S3，無 git 小檔」）。
- Source 載入透過 `pmtiles://` 自訂 protocol（由 `PmTilesSource` 實作 `CustomSourceType`），實際 `addSource` 呼叫散落在 `src/map/overlayRegistry.ts` 與各 `*LayerFactory.ts`（`agricultureLayerFactory.ts`、`fireIsochroneLayerFactory.ts`、`medicalIsochroneLayerFactory.ts` 等），本次未逐一列出每個 source id。

---

## 3. 第三方資料 API（TDX 等）

**結論：本 repo 前端沒有直接呼叫 TDX 或其他外部運輸資料 API 的程式碼。**

- 對 `src/` 全目錄 `grep -rn "fetch("` 逐一核對（見下表），所有非 `supabase.rpc` 的 `fetch()` 呼叫對象皆為：
  1. 相對路徑靜態資產（`./geo/*.geojson`、`./climate/*.json`、`/rail/tra/schedules_real/...`、`./h3/*.json` 等），由 nginx location 分流到 S3 或 dist fallback
  2. Supabase REST endpoint（`${SUPABASE_URL}/rest/v1/...`，見 §1.4、§1.3）
  3. `src/data/staticRpc.ts:25` 的 `/static-rpc/${name}.json`（同源靜態 CDN 快照）
  4. BYOK LLM 官方 SDK 的瀏覽器直連（見 §4，非「TDX 類」資料 API）
- 未發現任何 `tdx.transportdata.tw`、`flightradar24` 或其他外部網域字串出現在 `src/` 內。
- 依 recon.md 判定與 CLAUDE.md 資料源分工規則，**TDX / FlightRadar24 等原始資料抓取邏輯位於姊妹 repo `data-collectors`**（`scripts/fetch/*.ts` 雖在本 repo 但屬「資料收集腳本」，不隨 SPA bundle 出貨，執行環境是 Node script 而非瀏覽器，故與「前端外部整合」範疇區分）。
- 例外備註：`data.itsmigu.com`（Cloudflare R2 CDN，見 §5）與 CWA 影像相關，屬於「本方資料管線產出物的 CDN 直取」而非第三方資料 API，詳見 §5。

---

## 4. BYOK 多 LLM 聊天子系統（唯一真正的「第三方 API 瀏覽器直連」）

- `src/chat/providers.ts:1-34`：`createChatModel(provider, model, apiKey)` 依 provider 用 Vercel AI SDK 官方 factory 建立 model：
  ```ts
  case "anthropic":
    return createAnthropic({ apiKey, headers: { "anthropic-dangerous-direct-browser-access": "true" } })(model);
  case "openai":
    return createOpenAI({ apiKey })(model);
  case "google":
    return createGoogleGenerativeAI({ apiKey })(model);
  ```
  檔頭註解明講：「全部瀏覽器直連（BYOK），不經任何我方 proxy。key 只在此處交給官方 SDK，絕不落 log」（`src/chat/providers.ts:1-2`）。
- API key 來源：`src/lib/keyVault.ts`（BYOK 金鑰管理，本次未展開細節，依檔名與 recon.md 描述推測為 localStorage 存放使用者自帶金鑰）。
- 安全邊界靠 CSP `connect-src` 白名單收斂（見 §6），而非後端 proxy 攔截——這是本專案「BYOK 無 server-side 中介」架構的直接體現，key 外洩風險完全轉嫁給瀏覽器端 CSP 約束是否落實。

---

## 5. 失敗處理

### 5.1 Supabase RPC 失敗

- **標準模式**：loader 呼叫 `supabase.rpc(...)` 後檢查 `error`，非 null 就 `throw new Error(...)`，不吞錯，交由上層（React hook / `loadingRegistry`）呈現錯誤態。範例：`src/data/erHospitalLoader.ts:83-86`：
  ```ts
  const { data, error } = await withLoading("er:latest", "急診壅塞 59 院即時量能", supabase.rpc("get_er_hospital_latest"));
  if (error) throw new Error(`get_er_hospital_latest: ${error.message}`);
  return (data ?? []) as ErHospitalLatest[];
  ```
- **Retry**：不在 loader 層各自實作，而是集中在 `src/lib/supabase.ts` 的 `resilientFetch`（見 §1.1）—— 這是一個**跨所有 loader 的共用韌性層**，而非 per-loader circuit breaker。5xx/429/網路層錯誤最多 2 次 retry，寫入類 RPC（`log_session_events`）明確排除。
- **座標 join 類 loader 的降級**：`src/data/erHospitalLoader.ts:58-75` 的 `loadCoordMapUncached()` 用 `try/catch` 包住 geojson `fetch`，失敗時 `console.warn` 並回傳空 `Map`（而非 throw），讓急診資料仍可顯示但無座標（後續 join 邏輯會跳過 unmatched 項）——屬「部分降級」而非「全體失敗」。
- **靜態化 RPC fallback**：`src/data/staticRpc.ts:23-33` 的 `staticRpc()` 是專門設計的雙層 fallback：CDN 靜態 JSON 404/網路失敗 → 自動改打原本的 `supabase.rpc(name)`，`try { fetch } catch {} ` 吞掉網路層例外後 fallthrough，此設計讓「靜態檔案還沒部署」不會造成功能中斷，只是沒有加速效果（檔頭註解 `src/data/staticRpc.ts:12-13` 特別提醒驗收時要用 Network tab 確認真的打中 CDN 而非一路 fallback）。
- **普遍未見的機制**：**沒有 per-RPC 的 circuit breaker**（例如連續失敗 N 次後短暫停止呼叫某支 RPC）、**沒有 exponential backoff 之外的 jitter 策略差異化**（所有 RPC 共用同一組 backoff 常數）、**沒有離線 queue / background sync**。錯誤處理策略統一由 `resilientFetch` 一層兜底 + `loadingRegistry` 的錯誤態 UI 呈現（依 CLAUDE.md 規則 3，所有 loader 必須 `start()`/`complete()` 註冊 loading task，錯誤態於 UI 顯示但非本次追蹤重點）。

### 5.2 raw fetch（不經 supabase-js）失敗處理

- `src/data/satelliteLoader.ts:52-61` 的 `fetchView()` 僅檢查 `resp.ok`，`throw new Error` 帶 status code，**沒有 retry**（因未經 `resilientFetch`），失敗時上層依賴 `localStorage` 6h 快取（若快取存在則不受單次失敗影響，`readCache()`/`writeCache()`，`src/data/satelliteLoader.ts:34-50`）。
- `src/lib/sessionTracker.ts:60` 的 `flushViaBeacon()`：`.catch(() => {})` 完全靜默吞錯——這是刻意設計（分析事件遺失可接受，不應影響主體驗），與 §1.1 「不吞錯」原則的差異點在於：分析類旁路容忍靜默失敗，業務資料 RPC 不容忍。

---

## 6. 部署整合

### 6.1 Dockerfile（多階段建置）

- **Stage 1（build）**：`node:22-alpine`，`npm ci` → `npm run build`。Mapbox token **必須在 build time 注入**（因 Vite 靜態 inline env）：
  ```dockerfile
  ARG VITE_MAPBOX_TOKEN
  ENV VITE_MAPBOX_TOKEN=$VITE_MAPBOX_TOKEN
  RUN npm run build
  ```
- **Stage 2（serve）**：`nginx:alpine` + `apk add aws-cli`（S3 pull 用），複製 `nginx.conf`、`dist/`、三支 `scripts/deploy/*.sh`（`pull-deploy-assets.sh`、`refresh-climate.sh`、`entrypoint.sh`），`ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]` —— 容器啟動時先背景 pull S3 assets 到 `/data`，再啟動 nginx（pull 失敗不 crash，容錯設計）。

### 6.2 nginx.conf：S3 volume vs git-tracked dist fallback

模式分兩類（依各 `location` block 註解逐一核對，`nginx.conf` 全檔）：

| 類型 | Location 範例 | 行為 |
|---|---|---|
| **有 fallback**（大檔在 S3，小檔在 git dist） | `/geo/`、`/h3/`、`/bus/`、`/forestry/`、`/fishery/`、`/coverage/` | `root /data; try_files $uri @dist;` → 先找 `/data`（S3 volume），找不到轉 `location @dist { root /usr/share/nginx/html; }`（build 出的 dist，含 git-tracked 小檔） |
| **純 S3，無 fallback** | `/water_resources/`、`/agriculture/`、`/sports/`、`/static-rpc/`、`/medical/`、`/base_map/`、`/climate/`、`/fire/`、`/police_justice/`、`/road/`、`/rail/` | 僅 `root /data;`，資料完全不進 git（檔案過大或無需離線備援），S3 沒 pull 到就直接 404 |
| **Root 層動態資料** | `aviation_data.json`/`ship_data.json`/`temperature_grid.json` | Regex location `^/(aviation_data\.json\|ship_data\.json\|temperature_grid\.json)$`，同樣 `root /data` |

- `docker-compose.yml` 本地開發用 `volumes` 掛載對應到 `/data/*`（與 Zeabur volume 路徑一致），列出 15 個大檔（`aviation_data.json`、`ship_data.json`、`rail/`、`bus_stations_*.geojson`、`h3_demographics_res8.json` 等），供本地容器模擬生產環境 volume 結構。
- **PMTiles 透過 HTTP Range Request** 載入切片，nginx 預設支援，註解特別提醒「pmtiles 不在 gzip_types 內會自動 bypass」（`nginx.conf` `/fire/`、`/road/` location 註解）。

### 6.3 CSP（Content-Security-Policy-Report-Only）

`nginx.conf:171`（節錄，安全 header block）：
```
connect-src 'self'
  https://utcmcikhvxnohbxchbrs.supabase.co wss://utcmcikhvxnohbxchbrs.supabase.co
  https://api.mapbox.com https://events.mapbox.com https://*.tiles.mapbox.com
  https://api.anthropic.com https://api.openai.com https://generativelanguage.googleapis.com
  https://data.itsmigu.com
  https://pub-eae6980c040441adbd1eaded2870d3d2.r2.dev
```
- 目前為 **Report-Only** 階段（不阻擋，僅瀏覽器 console 記錄 violation），`nginx.conf:166-170` 註解說明切正式 enforcing 只需把 header 名稱從 `Content-Security-Policy-Report-Only` 改成 `Content-Security-Policy`。
- `connect-src` 白名單即是本文件 §1-4 整理出的所有外部整合對象的權威清單：Supabase（含 WebSocket）、Mapbox（API + tiles + events）、三家 BYOK LLM、以及 `data.itsmigu.com`（Cloudflare R2 自訂網域，CWA 衛星/雷達影像 CDN，見 `.claude/memory/BACKLOG.md` AR-11：「CWA 衛星/雷達影像上 R2 CDN…前端 `VITE_IMAGERY_CDN_BASE` flag 閘控」，對應程式碼 `src/data/cwaImageryLoader.ts:19`：`const IMAGERY_CDN_BASE = (import.meta.env.VITE_IMAGERY_CDN_BASE ?? "").replace(/\/+$/, "")`）+ R2 預設網域 `pub-eae6980c040441adbd1eaded2870d3d2.r2.dev` 作為備援/開發用網域。

### 6.4 CI（`.github/workflows/ci.yml`）

```yaml
on: { pull_request: { branches: [master] }, push: { branches: [master] } }
jobs:
  test:
    steps:
      - actions/checkout@v4
      - actions/setup-node@v4 (node-version: '20')
      - npm ci
      - npm run build   # tsc -b + vite build
      - npm test        # vitest
```
- 僅做**編譯期驗證（TypeScript project references `tsc -b`）+ Vite build + Vitest 單元測試**，**不含**：外部整合的 E2E/整合測試（無 Supabase/Mapbox mock 測試）、Docker image build 驗證、部署驗證。與 CLAUDE.md「commit 前必跑 `npx tsc -b` + `pnpm test`」的本地開發要求一致，CI 是同一組檢查的自動化版本。
- 另有 `claude-mention.yml`、`claude-review.yml`（recon.md 提及，未在本文件展開，屬 Claude Code 協作基礎設施而非外部資料整合）。

---

## 附錄：本文件未展開但與外部整合相關的延伸主題（供後續 Stage 參考）

- `src/lib/auth.ts`：Supabase Auth（OAuth）整合，`.claude/memory/BACKLOG.md` BC-4a 提及需在 Supabase Dashboard 手動設定 Redirect URLs + Google Console，屬「非程式碼可追蹤」的外部平台設定，本文件僅在 §6.3 CSP 脈絡中帶到。
- `src/lib/keyVault.ts`：BYOK API key 的 client-side 儲存機制細節，與 §4 LLM 整合直接相關但未展開實作。
- `src/map/overlayRegistry.ts` 與各 `*LayerFactory.ts` 中逐一的 PMTiles source id / layer id 對照表，屬「Layer 系統」而非「外部整合」範疇，建議留給 layer-onboarding 相關文件處理。
