# 食譜收藏夾 系統架構設計 (Architecture)

## 1. 技術架構說明

本專案採用傳統的伺服器端渲染 (Server-Side Rendering, SSR) 架構，以確保開發快速且系統穩定。

### 選用技術與原因
- **後端框架**：Python + Flask。Flask 輕量、靈活，適合快速開發 MVP，並且有豐富的套件生態能支援會員註冊與爬蟲邏輯。
- **模板引擎**：Jinja2。直接整合在 Flask 中，能有效地將後端處理好的資料注入 HTML 結構中，不需引入複雜的前端框架即可完成介面渲染。
- **前端樣式與互動**：HTML、Vanilla CSS、Vanilla JavaScript。為了保持專案輕巧，避免引入不必要的前端工具鏈與編譯步驟。
- **資料庫**：SQLite (搭配 SQLAlchemy 或內建 sqlite3)。無需額外啟動資料庫伺服器，資料輕便儲存於單一檔案中，非常適合初期開發與在地部署。

### Flask MVC 模式說明
在伺服器端渲染架構中，採用變形的 MVC (Model-View-Controller) 設計模式：
- **Model (模型)**：負責定義資料結構與資料表交互邏輯 (處理 `User` 驗證、`Recipe` 食譜資訊與食材關聯查詢等操作)。
- **View (視圖)**：Jinja2 模板與前端 HTML/CSS 靜態資源，負責從 Controller 接收變數並產生最終呈現畫面的 HTML。
- **Controller (控制器)**：Flask 的 Route 綁定，負責接收瀏覽器 Request（如新增食譜或搜尋食材），呼叫對應的 Model 修改或查詢資料，並將結果送交 View 進行渲染。

## 2. 專案資料夾結構

採用模組化結構來管理不同職責的程式碼，目錄配置如下：

```text
web_app_development/
├── app.py                 # 應用程式入口點，負責初始化 Flask 實例
├── requirements.txt       # 專案相依套件清單 (如 flask, bcrypt, requests, bs4 等)
├── docs/                  # 專案文檔目錄
│   ├── PRD.md             # 產品需求文件
│   └── ARCHITECTURE.md    # 系統架構設計文件 (本文件)
├── instance/              # 實體存放區，放置不進版控的本地資料庫或其他金鑰
│   └── database.db        # SQLite 資料庫檔案
├── app/                   # 主要應用程式邏輯目錄
│   ├── __init__.py        # 建立並配置 app 工廠函式
│   ├── models/            # Model 模組：資料庫 Schema 與物件包裝
│   │   ├── __init__.py
│   │   ├── user.py        # 處理帳號資料與密碼雜湊驗證邏輯
│   │   └── recipe.py      # 處理食譜、食材表，及多對多關聯操作
│   ├── routes/            # Controller 模組：Flask Blueprint 路由
│   │   ├── __init__.py
│   │   ├── auth.py        # 負責 /login, /register, /logout
│   │   ├── recipe.py      # 負責食譜的新增、編輯、搜尋與首頁列表
│   │   ├── admin.py       # 管理員專屬功能，如下架違規內容
│   │   └── crawler.py     # 負責處理 URL 輸入即產生解析食譜的爬蟲輔助路由
│   ├── templates/         # View 模組：HTML Jinja2 模板
│   │   ├── base.html      # 共用母版 (包含導航列、Footer、CSS 引用)
│   │   ├── auth/          # 登入與註冊頁面模板
│   │   ├── recipe/        # 食譜列表、詳細檢視、新增編輯之表單模板
│   │   └── admin/         # 後台管理介面模板
│   └── static/            # 靜態資源目錄
│       ├── css/
│       │   └── style.css  # 全域自訂樣式、深淺色系或大字體模式寫在裡面
│       └── js/
│           └── main.js    # 用於互動切換 (如「互動式烹飪模式」或非同步爬蟲請求)
```

## 3. 元件關係圖

以下展示核心操作流程，說明使用者從點擊到看到畫面的資料流與元件互動：

```mermaid
flowchart TD
    Browser[使用者瀏覽器]
    
    subgraph 伺服器端 Flask APP (Controller)
        Route_Auth[Auth Blueprint]
        Route_Recipe[Recipe Blueprint]
    end

    subgraph 模板渲染 (View)
        Jinja[Jinja2 Templates]
    end

    subgraph 資料層 (Model)
        Model_User[User Model]
        Model_Recipe[Recipe / Ingredient Models]
    end

    DB[(SQLite 資料庫)]

    %% 使用者發送請求流程
    Browser -- "HTTP Request \n (GET/POST 搜尋、發布食譜)" --> Route_Recipe
    Route_Recipe -- "ORM CRUD 操作" --> Model_Recipe
    Model_Recipe -- "SQL Query" --> DB
    
    %% 回傳資料流程
    DB -. "回傳關聯結果" .-> Model_Recipe
    Model_Recipe -. "包裝成 Python Object" .-> Route_Recipe
    Route_Recipe -- "傳遞物件變數給模板" --> Jinja
    Jinja -- "渲染轉換為 HTML" --> Route_Recipe
    Route_Recipe -- "HTTP Response (HTML)" --> Browser
```

## 4. 關鍵設計決策

1. **基於 Blueprint (藍圖) 劃分路由**
   - **原因**：將路由按功能拆分為 `auth`, `recipe`, `admin`, `crawler`，避免所有邏輯集中在 `app.py` 中造成維護困難。不僅可讓程式碼職責分明，也方便在各個 Blueprint 上加上一致的中介層（如 `@login_required`）。

2. **嚴密的安全性措施**
   - **原因**：為了防止常見資安風險，將採用 `bcrypt` 加密算法對使用者的輸入密碼進行單向雜湊；對於查詢資料庫一律使用 ORM (或防護良好的 sqlite3 參數化綁定) 阻絕 SQL Injection。再者，仰賴 Jinja2 預設的 Auto-escaping (自動跳脫 HTML 與 JS) 來防範 XSS 攻擊。

3. **優化查詢效能的 Schema Design**
   - **原因**：PRD 要求支援「多種食材組合的食譜篩選」，如果採用字串比對 (`LIKE`) 效能低落。為此，在 Model 設計上採用正規的關聯表 (Join Tables) 將 Recipe 與 Ingredient 拆分，並對於成對的外鍵欄位建立複合索引 (Composite Index)，以便快速執行 SQL 的交集 (`INTERSECT`) 查詢，大幅度減少運算負擔。

4. **採用 Vanilla JS 漸進式增強**
   - **原因**：在渲染流程中以 Jinja2 為主，但對於「一鍵網頁抓取」、「食材清單勾選」與「互動烹飪模式切換」等不需重新載入頁面的操作，運用 Vanilla JavaScript 搭配 Fetch API 送出異步請求 (AJAX)。這樣兼顧了 SSR 開發快速的優點，也具備現代動態網頁的好體驗。
