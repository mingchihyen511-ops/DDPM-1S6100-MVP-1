# 🚀 Project DDPM UI Automation Framework (MVP-1)
## 📋 系統設計、執行架構暨預期成果說明書 (規格驗收版)

本文件完整記錄了截至 2026 年 6 月 10 日，針對 DDPM UI 專案（如版號 DDPM_2.4 核心設計稿）所開發之自動化規格稽核與監控系統（MVP-1）的完整架構。旨在作為跨視窗移轉、CI/CD 部署、以及團隊技術對齊（Alignment）之唯一黃金標準憑據。

---

## 🎯 一、 專案核心初衷與高階目標 (Purpose & Target)

在大型軟體專案開發中，UX 設計稿的頻繁更動常導致開發工程師（RD）規格出錯。本專案（MVP-1）的核心目標在於建立一套**自動化增量監控與反解析對撞閉環系統**：
1. **實時稽核設計變更**：透過計時器與 Checksum 機制，自動捕捉設計師在 Figma 畫布上的規格與狀態修改，自動生成 Git-Style 的繁體中文異動 Diff 報告。
2. **自動組織分流派案**：依據 Figma 分頁標記，自動匹配通訊 Roster 表中的 Domain 負責人，實現精準的郵件/通知派案。
3. **高擬真 UAT 反解析驗收**：確保本地下載的數據結構飽滿，能被 100% 離線反解析還原為具備圖層、文字、顏色、組件狀態與物理位置（Bounding Box）的實體圖，供設計師進行 1:1 調試與工業級驗收。

---

## 🛠️ 二、 三大核心組件執行架構 (System Components)

系統完全捨棄了繁瑣的手動設定檔（Zero-Config），由以下三大模組相互嚙合運作：

### 1. 📡 智慧分片 Indexer 引擎 (`FigmaPageIndexer.py`)
負責將幾十萬圖層、過於肥大（>23.7 MB）的單一 Figma JSON，在下載端實施「動態物理阻斷與硬拆分」，徹底繞過 GitHub 25MB 上傳紅線。
* **免設定·動態領域拆分**：由上而下掃描 Figma Page 列表。一旦撞到 `💠` 或 `◆` 開頭的分頁（如 `💠 MOUSE`、`💠 KEYBOARD`），指針自動切換，將後續分頁歸類並落盤為乾淨的個別小 JSON（如 `MOUSE.json`）。
* **`OTHERS` 兜底機制**：在遇到第一個 `💠` 標記前出現的任何有效設計頁，一律自動歸入 `OTHERS.json`，100% 阻斷漏頁風險。
* **斷點式即時儲存 (Checkpoint Save)**：不再等待全專案下載完。每爬完一組 `💠` 領域，立刻啟動 8 路多執行緒並發下載，下載完瞬間調用 `json.dump` 安全落盤。若後續模組因網路超時崩潰，先前落盤的微快取絕對不流失，並支援重跑時 0.1 秒秒級續傳。
* **精準過濾排他鏈**：完全排除 `❌` 開頭的封存分頁、純橫線裝飾線、`Cover` 與 `Prototypes` 測試頁。**保留 `Translation` 多國語系對照頁**。
* **組件實例深潛 (Instance Penetration)**：打破限制，允許大腦深潛進入 `"INSTANCE"`、`"COMPONENT"`、`"COMPONENT_SET"` 容器中，100% 擷取藏在元件最深處的 TEXT 實質文字（如 `HDR`, `Auto White Balance`）與幾何座標。

### 🧠 2. 循序對撞比對大腦 (`FigmaMonitorAgent.py`)
負責對比歷史微快取 Baseline 與雲端實時數據，進行語意級對沖，並產出高導航性的 Diff 報告。
* **全域閘門雜湊比對**：優先撈取雲端 depth=1 的不可變 `version` (如 Git Commit Hash) 與 `lastModified` 時間戳。若一致則 0.5 秒直接 Bypass 退出；若不一致則輸出詳細的版號與物理時間差異診斷，並啟動比對。
* **模組級循序對撞技術**：拒絕 73 個頁面混亂併發。程序依 Domain（模組）循序遍歷，限制單一模組執行緒池（max_workers=5）。終端機日誌結構清晰美觀、對 Figma API 極具親和力。
* **語意角色識別器與剪枝**：自動將組件角色分類為 `TOGGLE` (開關: 辨識 ON/OFF 狀態)、`SLIDER` (滑桿: 擷取 50% 等數值與主題色彩)、`BUTTON_TAB` (分頁按鈕組: 擷取選中高亮狀態) 兩大類。一經判定立刻剪枝（Pruning）終止下潛，保持結構極致乾淨。
* **測試模式安全鎖 (TEST_MODE)**：設為 `True` 時，僅於 `report/` 輸出 txt 異動報告，100% 保護黃金 Baseline 不被髒化。

### 🖥️ 3. 離線反解析驗證看板 (`FigmaUatVisualizer.py`)
基於 **Approach B (本地快照反解析)** 實作的 1024x720 雙面板 Tkinter 圖形化介面（GUI），100% 離線運行，用以驗證快照的特徵飽滿度。
* **手動填寫與智慧清單彈窗**：啟動時自檢 Page 目錄環境。通過後彈出對話框，支援手動填入 Node ID，或雙擊系統在快照中自動網羅出的所有實體 UI 元件清單一鍵載入。
* **絕對視口原點錨定 (Root Anchor)**：以所選 Node ID 自身 bounding_box 為主原點縮放，配合 **Canvas 實體滾動條 (Scrollbars) 與 Zoom 下拉選單 (25%~150%)**，徹底解決大座標導致的重疊與跑位硬傷！
* **一鍵過濾結構容器 (Show Containers Toggle)**：預設**不勾選**。隱藏幾十層 FRAME 的無效虛線框與灰色標籤，畫布只純淨渲染出：Toggles、Sliders、Tabs 與實體規格文字（文字直接原位渲染），精度 1:1 直逼 Figma 實體設計圖。
* **富文本屬性面板 (Property Inspector)**：滑輛懸停亮起金黃框、點擊亮起紅色鎖定框，右側面板實時以 Consolas 字體解析出座標尺寸、字型文字、Figma 變體等全套 Token 供人工稽核。

---

## 🔍 三、 UAT 驗收標準與資料定義 (Data Schema)

為確保每週版本比對的絕對精確，快照 JSON 在反解析時必須滿足以下四維度規格：

1. **圖層佈局 (Layout)**：記錄 `Node ID` 雜湊鍵與 `type` (如 `INSTANCE`)。不論設計師如何在畫布上挪移位置，`Node ID` 永恆不變，確保 Git 比對線條乾淨。
2. **文字規格 (Texts)**：封裝 `text_content` 內容，搭配【語意級麵包屑路徑】（如 `[Earbud Controls ➔ WL327 ➔ Tooltip]`），使 RD 能在代碼中原位追溯。
3. **主題色彩 (Colors)**：智慧分析 fills 向量，跳過黑白灰，抓出開關/滑桿的實質主題色彩（如 Dell 科技藍 `#007BFF`）。
4. **元件物理幾何 (Geometry)**：同步封裝 `absoluteBoundingBox` 的 `x, y, width, height`。當位移大於 **`1.0 px` 智慧閥值**時，比對大腦會發出 layout shift 警告並算出 $\Delta X$ 與 $\Delta Y$，**強力攔截因跨專案複製貼上 Constraints 失效導致的「F1~F12 按鍵框跑位」重大故障！**

---

## 🚀 四、 局部沙盒 UAT 測試驗收手冊 (Step by Step)

為了達到極速 UAT 對位與調試，請遵循以下「縮小任務、局部驗收」步驟：

1. **鎖定測試白名單**：
   打開 `FigmaPageIndexer.py` 頂部的 `PAGE_WHITE_LIST` 設定欄位，填入您要測試的專屬分頁關鍵字：
   ```python
   PAGE_WHITE_LIST = ["Blue Moon WB126"] # 亦可填入 "Earbud Controls", "WH327" 等
   ```

---

## ⚙️ 五、 全專案化與併發優化修正計畫 (Approved Plan)

本階段將 MVP-1 從 WB126 局部測試推進到 73 pages 全專案流程。優先目標不是裁剪資料，而是透過更好的併發調度縮短全量掃描時間，同時保留快照資料飽滿度：圖層、文字、顏色、字型、座標、元件狀態與 bounding box。`FigmaUatVisualizer.py` 保留為抽樣 UAT 工具，不納入每頁主流程。

### 1. `FigmaPageIndexer.py` 修正方向

* **全專案預設掃描**：移除 WB126 作為主流程限制，預設掃描全專案 73 pages。
* **白名單保留為沙盒入口**：`PAGE_WHITE_LIST` 改為可選測試模式，用於局部驗證或問題頁快速重跑，不再作為正式流程預設。
* **快照 Schema 補強**：在既有 `text_content`、`ui_role`、`ui_color`、`variants`、`bounding_box` 基礎上，補齊 TEXT / style 字型資訊，例如 font family、font size、font weight、line height、letter spacing、fills。
* **併發調度優先**：調整跨 page 下載與解析策略，以受控 worker pool 同時處理多頁，縮短 73 pages 全量生成時間；同時保留 retry/backoff，避免 Figma API 429 限流。
* **既有分片規則保留**：保留 Domain 分片、checkpoint save、`OTHERS` 兜底、Translation 保留、封存/測試頁排除、INSTANCE / COMPONENT / COMPONENT_SET 深潛解析。

### 2. `FigmaMonitorAgent.py` 修正方向

* **移除固定 WB126 測試鎖**：正式流程改為全專案監控，不再寫死單一 page。
* **保留全域閘門**：先以 Figma `version` / `lastModified` 比對雲端與本地 baseline；若未變更，快速重用 baseline。
* **變更時分批並發下載**：若全域 checksum 顯示雲端已變更，依 Domain 分批並發下載 active pages，再與本地 baseline 對撞。
* **測試安全鎖維持**：`TEST_MODE=True` 時只輸出 `report/` txt，不覆寫 `Page/` baseline、不發信。
* **Diff 報告補強**：報告需能呈現文字、狀態、名稱、座標位移與重要 style 變更，並維持繁體中文 Git-style 可讀格式。

### 3. `FigmaUatVisualizer.py` 定位調整

* **抽樣驗收工具**：Visualizer 僅作為高風險頁面、diff report 指定 Node ID、或設計師抽查時使用，不要求每頁人工檢測。
* **支援新 Schema**：右側 Property Inspector 需讀取並顯示新 snapshot schema 的字型/style 欄位。
* **主流程不依賴 GUI**：CI/CD 或日常監控流程不得依賴 Tkinter GUI 啟動；GUI 只服務人工 UAT。

### 4. 快照資料介面定義

Snapshot page node 至少保留以下欄位：

* `node_id`
* `name`
* `type`
* `ui_role`
* `text_content`
* `fills` 或 primary color
* `text_style`
* `bounding_box`
* `variants`
* `ui_state`

Baseline 仍維持多檔分片結構：

* `Page/<project_alias>/metadata.json`
* `Page/<project_alias>/<domain>.json`

### 5. 驗收測試標準

* **全專案初始化**：確認 73 pages 可成功生成分片 JSON，且單一 JSON 不超過 GitHub 25MB 上傳限制。
* **效能觀測**：記錄總耗時、每 Domain 耗時、每 page 耗時、API retry 次數與 429 次數。
* **Schema 抽查**：抽查 TEXT 節點必須包含文字、字型/style、座標與顏色資訊。
* **Monitor 快速閘門**：雲端未變時應快速 bypass 或重用 baseline，不重新下載全量 page nodes。
* **Monitor 變更報告**：雲端有變時產出繁體中文 diff report，且 `TEST_MODE=True` 不覆寫 baseline。
* **UAT 抽樣**：使用 diff report 中的 Node ID 開啟 Visualizer，確認位置、文字、顏色與元件狀態可反解析。

### 6. 實作假設與預設

* 第一版以「並發調度優先」，不先犧牲資料完整度換速度。
* 第一版目標是全專案化，WB126 僅保留為可選沙盒測試。
* Figma API 限流風險需透過 worker 上限、retry/backoff 與日誌統計處理。
* Visualizer 是輔助驗收工具，不是 CI/CD 主流程必要步驟。
