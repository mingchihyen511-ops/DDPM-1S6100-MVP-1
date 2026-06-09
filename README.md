# DDPM-1S6100-MVP-1

DDPM-1S6100-MVP-1: Figma 跨版本 UI 變更自動化監控系統 
🚀本專案為 Figma 跨版本 UI 變更自動化監控系統 的第一階段實作
（MVP-1：設計端每週變更監控系統 - Design Monitor）。
透過自動巡邏與三層漏斗過濾演算法，本系統能夠在極低 API 流量消耗的前提下，精準捕捉 Figma 設計稿的目錄異動、頁面文字修正、開發狀態切換，並自動對應組織分流，派發變更報告給負責的設備開發小組。

📌 目錄 (Table of Contents)
一、 環境準備 (Environment Preparation)
  1. 安全金鑰與環境變數管理
  2. 本地目錄結構初始化
  3. Python 依賴套件安裝
二、 運行程序與架構 (Execution & Architecture)
三、 核心模組與開發步驟 (Development Steps)
四、 測試步驟 (Testing Steps)
五、 上線驗收步驟 (Validation Steps - UAT)
六、 潛在風險與防範對策 (Risks & Mitigations)
=============================================================

一、 環境準備 (Environment Preparation)本系統預期於 GitHub Actions (GitHub Runner) 雲端排程環境中執行，同時支援本地 VS Code + GitHub Copilot 開發偵錯環境。
1. 安全金鑰與環境變數管理本地開發 (.env)：在專案根目錄下建立 .env 檔案。
2. 此檔案已寫入 .gitignore，嚴禁推上 GitHub。FIGMA_TOKEN=figd_你的個人訪問權杖

# 本地測試郵件網關設定
SMTP_SERVER=smtp.company.com
SMTP_PORT=25
GitHub 雲端儲存庫 (Secrets)：在 GitHub 儲存庫的 Settings -> Secrets and variables -> Actions 中，新增以下加密金鑰（Secrets）：FIGMA_TOKEN：Figma API 訪問權杖。SMTP_SERVER / SMTP_PORT：公司內部專用 Mail Gateway 設定。

2. 📁 本地目錄結構初始化系統執行前，工作目錄需具備以下解耦結構：E:\DDPM\
│  .env                        # 本地安全配置 (排除於 Git 之外)
│  .gitignore                  # Git 排除清單 (.env, __pycache__, Page/*.json, Log/*.json)
│  files_dict.json             # 【主字典】純專案與領域範疇定義
│  team_roster.json            # 【人事表】領域負責人、代理人與全域 CC 清冊
├───.github\
│   └───workflows\
│           figma_monitor.yml  # GitHub Actions 自動每週排程定義檔
├───Page\                      # 【快照目錄】Agent 自動寫入之結構 JSON (Git 追蹤)
└───Log\                       # 【日誌目錄】Agent 自動寫入之異動流水帳 (Git 追蹤)

📄 主字典檔案樣例：files_dict.json本檔案由人類手動維護，僅定義 Figma 目錄之領域歸屬，完全不碰人名與 Email：{
  "projects": {
    "DDPM_2.4": {
      "file_key": "ngpoWBPQHsJp1vglCBg0kw",
      "domains": {
        "AUDIO_DOMAIN": {
          "domain_marker": "◆ AUDIO",
          "pages": [
            "Landing Page", 
            "Audio Settings", 
            "Earbud Controls", 
            "Automated Actions",
            "[WIP] Walkthrough",
            "Archive_Old_Audio_v1"
          ]
        },
        "TRAVEL_HUB_DOMAIN": {
          "domain_marker": "◆ TRAVEL HUB",
          "pages": ["Overview", "Connection Settings", "Archive_2025_Backup"]
        }
      }
    }
  }
}
👥 組織通訊錄樣例：team_roster.json本檔案負責組織與動態代理人矩陣，成員異動與代理僅需修改此表，不影響核心主字典與程式邏輯：{
  "domain_assignments": {
    "AUDIO_DOMAIN": {
      "owner": "wayne@email.com",
      "delegates": ["kevin@email.com"],
      "description": "負責 AUDIO 分類下屬所有有效 Pages 異動（自動過濾 WIP/Archive）"
    },
    "TRAVEL_HUB_DOMAIN": {
      "owner": "kevin@email.com",
      "delegates": [],
      "description": "負責 TRAVEL HUB 分類下屬所有設計開發"
    }
  },
  "global_management": [
    "alex.yen@wistron.com",
    "bruce@email.com"
  ]
}

3. Python 依賴套件安裝建立 requirements.txt 並安裝以下標準依賴套件（盡可能採用內建模組以降低相依性與資安風險）：python-dotenv==1.0.1
requests==2.32.3
本地安裝指令：pip install -r requirements.txt

二、 運行程序與架構 (Execution & Architecture)當每週排程（或手動排程任務）啟動時，系統的資料流與決策邏輯如下圖所示（此圖在 GitHub 上將自動轉譯為動態流程圖）：graph TD
    Start([GitHub Actions / 手動啟動]) --> ReadConfig[1. 環境與設定讀取<br>載入 Secrets, files_dict.json, team_roster.json]
    ReadConfig --> FilterFile{2. 第一層過濾: File 級<br>比對 lastModified & version}
    
    FilterFile -- 無變更 🟢 --> EndNoChange[直接結束排程<br>不發信、不覆寫快照]
    FilterFile -- 有變更 🟡 --> FilterPage{3. 第二層過濾: Page 級<br>比對各頁面 version_id}
    
    FilterPage -- 帶有 Archive 標記 ❌ --> SkipArchive[🛡️ 防火牆攔截: 直接忽略]
    FilterPage -- 帶有 WIP 標記 🚧 --> LogWIP[🛡️ 防火牆攔截: 僅記錄狀態於快照<br>不抓取實質內容、不比對]
    FilterPage -- 正常 Page 且有變更 🔴 --> FilterNode{4. 第三層過濾: Node 級<br>局部呼叫 nodes API 遍歷樹狀圖}
    
    FilterNode -- 節點名稱帶有 Concept / Do not use ⚠️ --> SkipNode[🛡️ 阻斷下屬子層遍歷<br>僅將狀況記錄於清單中]
    FilterNode -- 正常有效區塊 ✅ --> DiffCalc[計算實質文字差異 Diff<br>覆蓋本地舊快照 JSON]
    
    DiffCalc --> Route[5. 派案與分流發信]
    Route --> FindDomain[對照主字典: 識別變更屬於哪個 Domain]
    FindDomain --> FindTeam[對照通訊錄: 找出 Owner 與 CC 代理人]
    FindTeam --> SendMail[📬 寄出精準 Diff 異動信件<br>並追加紀錄於 changelog.json]

三、 核心模組與開發步驟 (Development Steps)
本系統開發流程分為六個核心階段，循序漸進實作：
🧱 Step 1: 環境配置載入器 (Config Loader)目標：讀取並驗證環境安全金鑰，加載 files_dict.json 與 team_roster.json。開發重點：當 FIGMA_TOKEN 為空時，拋出具備明確設定指南的 Exception 並終止程式。對兩份 JSON 進行 Schema 語意格式安全檢查，防止人類手動維護格式出錯導致程式崩潰。
🧬 Step 2: 檔案級與頁面級比對引擎 (File & Page Matcher)目標：利用 depth=1 輕量化 API，進行漏斗的第一、二層過濾。開發重點：比對雲端與本地快照的 lastModified 與 version。遍歷比對 Page 節點的 version_id 與 devStatus。
🛡️ Step 3: 目錄命名防火牆 (Directory Firewall Filter)目標：實作 WIP、Archive 標記字串自動過濾。開發重點：使用正則表達式或字串檢查判定 Page 名稱。偵測到 Archive（不限大小寫）➔ 直接將該 Page 從記憶體比對清單中剔除（Exclude），節省流量。偵測到 [WIP] ➔ 將 Page 快照標記為 WIP，不下載其下子節點，不進行內容級衝突檢查。
🔍 Step 4: 節點級局部下載與 Concept 攔截器 (Node Traverser & Concept Blocker)目標：局部調用 /v1/files/:key/nodes?ids=:page_id 下載變更頁面。開發重點：實作廣度優先（BFS）或深度優先（DFS）遍歷演算法尋找 Section / Frame / Text 節點。當節點名稱包含 Concept、Do not use ➔ 立即切斷對該節點以下所有子層級（Children）的遍歷，將其資訊寫入 ignored_nodes_status 陣列，不抓取實質內容，保護設計師草稿沙盒。
📝 Step 5: 文字差異比對模組 (Text Diff Engine)目標：比對新舊 Node 下屬的 text 屬性，產出規格化的差異。開發重點：比對相同 Node ID 內的 text_content。產出標準的 Git Style Diff（- 舊文字、+ 新文字）。
📬 Step 6: 派案分流發信引擎 (Mail Router)目標：將 Diff 異動與 Domain 負責人對撞，精準派信。開發重點：根據變更 Page 對照 files_dict.json 的 domains，抓出對應的 Domain_ID。用 Domain_ID 當 Key 去查 team_roster.json，取得 owner 信箱作為 To，delegates 信箱作為 CC。加上 global_management 中的長官信箱作為全域 CC。

四、 測試步驟 (Testing Steps)
為確保系統在實際上線前穩定運行，開發者需在本地或測試分支完成以下單元與情境測試：
🧪 1. 單元測試 (Unit Tests)T1 - 密鑰驗證測試：故意清空 .env 中的 Token，確認程式是否精確攔截並發出錯誤警告。T2 - 目錄防火牆測試：撰寫 Mock Page List，包含 "Audio Settings"、"Archive_Audio"、"[WIP] Audio"。預期結果：Archive 頁面需完全不被載入；WIP 頁面僅建立空殼結構、不下載內容。T3 - Concept 阻斷測試：建立 Mock 樹狀節點，包含一個名為 "[Concept] EQ Control" 的 Section 及其子節點。預期結果：該 Section 下屬的子文字節點在快照中均為空值，且 ignored_nodes_status 清單中成功記錄該 Concept 狀況。
👥 2. 組織與代理人異動測試 (Org Routing Test)T4 - 主要負責人異動：在 team_roster.json 中將 AUDIO 負責人由 Wayne 的信箱改為測試信箱。預期結果：異動信件成功改寄至測試信箱。T5 - 臨時代理人與 CC 測試：在 team_roster.json 代理人中加入多位成員。預期結果：產生的郵件標頭（Mail Headers）中，To 只有主要負責人，CC 包含了所有的代理人與全域 CC 成員。

五、 上線驗收步驟 (Validation Steps - UAT)上線驗收（User Acceptance Test）必須模擬一整套跨週實戰，確保資料流閉環。
📅 Day 1: 初始同步驗證執行 FigmaPageIndexer.py。檢查 Page/ 資料夾下是否自動產生對應 File Key（如 ngpowbpqhsjp1vglcbg0kw.json）的快照檔案。打開快照 JSON 驗證：lastModified 與 version 是否成功從 Figma 雲端撈回。被忽略的 Archive 是否未出現在 pages 中。名稱帶有 WIP 的頁面，其底下的內容節點是否完全被防火牆攔截（為空值）。
📅 Day 8: 跨週異動與精準派信驗收設計師模擬改稿：請 UX 設計師在 Figma 上，針對 Audio Settings 頁面下的 WL527 型號，修改 Tooltip 內的提示文字。手動/自動觸發監控：執行 FigmaMonitorAgent.py。驗證產出與信件：快照驗證：檢查本地快照 JSON 中的 lastModified 是否更新為最新時間，Tooltip 內的文字是否已被覆蓋為最新內容。日誌驗證：檢查 Log/changelog.json 是否自動追加了一筆變更紀錄，內容詳細載明 WL527 Tooltip 的 Diff。信件驗證：檢查 Wayne 的郵件，確認其信件內文「僅包含」Audio Settings 異動，沒有收到其他無關模組的通知。長官與代理人驗收：確認 Alex、Bruce 與 Kevin（Wayne 的代理人）是否在該信件的 CC 名單 中。
🔁 Day 9: 重複巡邏零垃圾警報驗證再次執行 FigmaMonitorAgent.py（此時 Figma 雲端無任何改動）。驗收指標：程式必須在第 1 層（File 級比對）秒級結束。本地快照 JSON 與 Changelog 不能有任何修改（No Dirty Commits）。任何人不得收到任何郵件（零 False Alarm 垃圾警報）。

六、 潛在風險與防範對策 (Risks & Mitigations)
在大型跨國團隊的網路與協作環境下，系統實際上線會遇到以下實務挑戰：
  🚨 1. Figma API 限流風險 (Rate Limit - 429 Error)實務情境：當多人同時開發，或是 GitHub Actions 設定每 5 分鐘就跑一輪，Figma 雲端會判定為 DDoS 攻擊並暫時封鎖 API 請求。對策：三層漏斗過濾：絕不進行高頻下載。在 File Level 比對即結束 90% 的請求。呼叫間隔延遲 (Backoff)：在 Python 請求機制中引入 time.sleep(1)，避免短時間內對 Figma 伺服器發射密集的局部 nodes 請求。
  🚨 2. Windows 與 Linux (GitHub Runner) 檔案大小寫衝突實務情境：Figma 產生的 File Key 有時包含大小寫英文字母。在 Windows 環境下，檔名大小寫不敏感；但 GitHub Actions 運作於 Linux 環境，大小寫極度敏感，會產生兩個獨立檔案，造成同步與 Git 比對錯亂。對策：系統在存取 Page/{FILE_KEY}.json 時，一律強制使用 .lower() 將檔名轉為純小寫，徹底消滅跨作業系統的檔案命名衝突。
  🚨 3. Mail Gateway 擋信與垃圾郵件過濾 (Spam Filter)實務情境：由 GitHub Actions (來自 AWS/Azure 浮動 IP) 直接發射的自動化信件，會被公司內部資訊安全的 Mail Gateway 直接阻擋或判為垃圾信。對策：對策 A：發信元件不使用公網 SMTP，而是透過授權的公司內部 Mail API 或專屬中繼閘道進行轉發。對策 B：郵件標題（Subject）必須遵循固定格式（如 [DDPM UI Change Alert]），並在內文中加上合規的自動化腳本說明，以利資安審查。
  🚨 4. Figma 雲端偶發性同步延遲 (Race Condition)實務情境：設計師網頁版 Figma 剛關閉，Figma 雲端伺服器還在進行多地備份（Sync），此時排程啟動，抓到的 lastModified 已更新，但實質 nodes 內容卻還是舊的，導致 Checksum 誤判。對策：當發現 lastModified 與本地不一致時，Agent 可稍微等待數秒後才進行第三層節點獲取，或者在 Changelog 驗證中，增加「二次 Checksum 認證」，確保下載下來的內容與 version_id 完全相符。
  🚨 5. 巢狀 JSON 深度爆炸 (Stack Overflow)實務情境：UX 設計稿若畫得極度複雜，畫布內有 Frame、Group、Section 多達十幾層巢狀，遞迴搜尋演算法可能導致 Python Stack overflow。對策：限制搜尋深度上限（例如 MAX_DEPTH = 15），一旦超過此深度不再往下遍歷。優先採用迭代式（Iterative）BFS 廣度優先搜尋，徹底防範遞迴深度造成的記憶體崩潰。
