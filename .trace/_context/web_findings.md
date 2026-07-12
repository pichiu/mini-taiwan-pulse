# Web Findings（Stage 1.5 線上資源搜尋）

> ⚠️ 本專案（Mini Taiwan Pulse）為私有內部專案，無公開官網 / Wiki / Discussion，故本文件聚焦於**專案整合的第三方平台與技術棧的官方文件**，而非專案本身的線上資源。

## 1. TDX 運輸資料流通服務平台（交通部）

專案多個圖層（公車、路況、停車、台灣好行）疑似串接 TDX 平台資料（依 `.claude/memory` 中「台灣好行」「路況」「停車」等描述推測，實際 collector 端點需在 Stage 2 `integrations.md` 以程式碼驗證 ⚠️ 未驗證）。

- 官方入口：[TDX 運輸資料流通服務](https://tdx.transportdata.tw/)
- Sample code：[tdxmotc/SampleCode (GitHub)](https://github.com/tdxmotc/SampleCode)
- 介接指南：[TDX 運輸資料介接指南 (bookdown)](https://bookdown.org/chiajungyeh/TDX_Guide/)
- API 使用限制：每呼叫來源 IP 每秒最多 50 次；訪客帳號每日基礎資料服務 20 次，會員每月合計約 3,000 次。

## 2. Supabase（後端 BaaS，含 PostgREST + pg_cron + PostGIS）

- Timeout 官方文件：[Supabase Docs - Timeouts](https://supabase.com/docs/guides/database/postgres/timeouts) — 全域 timeout 2 分鐘，這與 CLAUDE.md 提到的「Supabase pooler 強制 2min timeout 不能繞，只有 pg_cron 例外」一致，佐證 §4 pre-aggregate pattern 存在的必要性。
- pg_cron debugging：[pg_cron debugging guide](https://supabase.com/docs/guides/troubleshooting/pgcron-debugging-guide-n1KTaz) — pg_cron 支援最多 32 個並行 job，各佔一條 DB connection（對應 PRINCIPLES.md 「cron 錯開分鐘」教訓）。
- PostgREST 版本說明：[PostgREST v11.1 release notes](https://supabase.com/blog/postgrest-11-1-release)

## 3. Mapbox GL JS + Three.js（前端渲染核心）

- Mapbox GL JS 為業界標準向量地圖引擎，支援 `CustomLayerInterface` 讓開發者掛載自訂 WebGL/Three.js render pipeline — 對應本專案 `src/map/*CustomLayer.ts` 的技術基礎。
- Three.js 作為疊加在 Mapbox WebGL context 上的 3D 渲染函式庫（scene/camera 需與 Mapbox 的 mercator 投影對齊）。
- ⚠️ 未進一步搜尋兩者確切版本用法差異，因專案內部 `.claude/pitfalls/` 與 `PRINCIPLES.md`（L824「一個 Mapbox gl context 只掛一個 Three.js CustomLayer 實例」）已記錄專案特定教訓，優先度高於通用網路資料。

## 4. 為何本次未做更廣泛搜尋

- 專案是私有 monorepo 生態系一部分（另有 gis-platform / taipei-gis-analytics / data-collectors / pulse-api 姊妹 repo），這些內部知識無法透過公開 web search 取得，必須靠 Stage 2 程式碼追蹤與既有 `.claude/memory/` `docs/features/*` 取得。
- 專案已有極豐富的既有文件（151 個 `docs/*.md`），比對既有文件與程式碼落差（DISCOVERY_LOG.md）比額外網路搜尋更具產出價值。
- 依 Stage 1.5 判準（小型/內部專案不必硬搜），本次搜尋控制在 2 次，聚焦於「無法從程式碼本身推得的外部限制」（API rate limit、timeout 上限）。
