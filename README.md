# Automated Carbon Accounting & Human-in-the-Loop Audit System
### 基於 n8n 與 Python 的自動化溫室氣體盤查（Scope 1-3）計算與人機協作（HITL）審核系統

[![n8n](https://img.shields.io/badge/Workflow-n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io/)
[![Docker](https://img.shields.io/badge/Container-Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Python](https://img.shields.io/badge/Language-Python%203.10%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![Standard](https://img.shields.io/badge/Standard-GHG%20Protocol%20%7C%20ISO%2014064--1-007A3D?style=for-the-badge)]()
[![Architecture](https://img.shields.io/badge/Design-Human--in--the--Loop%20(HITL)-0052CC?style=for-the-badge)]()

---

## 專案緣起與核心痛點 (Background & Problem Statement)

本專案為**國科會產學研究計畫**之研究成果。隨著國際供應鏈對 ESG 永續標準與淨零碳排要求日趨嚴格，製造業與中小企業（SMEs）在推動溫室氣體盤查時面臨三大難題：

1. **營運資料高度碎片化與格式異質**：活動數據分散在廠務用電、ERP 出貨單、採購憑證、商務差旅及廢棄物清冊中，手動收集耗時數週且易遺漏。
2. **範疇三（Scope 3）計算複雜度高**：涉及採購材料、上下游物流運具、資本財等多種方法論（質量法、支出法、噸公里法），中小企業缺乏專責永續系統支援。
3. **缺乏客觀稽核與防呆防線**：傳統試算表缺乏數據軌跡記錄；若採全自動化又容易因異常數據灌入而失真，缺乏關鍵的「人機協作覆核」機制。

本系統建構了一套**容器化、輕量且具備審查稽核軌跡的解決方案**：以 Python 客戶端自動批次整合 11 項營運資料集，調用以 Docker 運行的 n8n 自動化流程引擎，依據 **GHG Protocol** 與 **IPCC AR6** 係數庫進行範疇一至範疇三的自動化運算，並利用 **P95 統計檢定** 自動辨識異常峰值月份，最終封裝四份專業清冊為審核套件（`review_bundle.zip`），透過 Gmail API 自動派案給永續主管進行簽核。

---

## 📁 專案結構

| 檔案名稱 | 說明 |
| :--- | :--- |
| `自動化碳盤查排放量+人工審核.json` | n8n 主工作流程檔，負責數據解析、GHG 運算、ZIP 打包與 Gmail 自動派案 |
| `碳盤查.ipynb` | 資料前處理、欄位映射驗證與 Webhook 呼叫測試 Jupyter Notebook[cite: 2] |
| `碳盤查_scope1.csv` | 範疇一活動數據（製程燃料燃燒、冷媒逸散補充） |
| `碳盤查_scope2.csv` | 範疇二活動數據（外購電力消耗量、綠電扣抵比例） |
| `碳盤查_scope3.csv` | 範疇三活動數據（原料採購、上下游運輸物流、差旅、通勤、廢棄物） |
| `碳盤查數據集_總表.csv` | 跨期活動數據彙整總表 |

---
## 📊 成果展示：自動化派案與人工覆核通知信 (HITL)

當 n8n 完成全量資料清洗與 GHG Protocol 運算後，系統會自動生成各範疇月度數據表、範疇三熱點分析，並透過 Gmail API 自動寄出稽核覆核信件給審查專員：

### 審查信件實際成果

> **郵件主旨**：`[碳盤查覆核通知] ACME Ltd. 2024 年度排放量審核資料`
<img width="742" height="689" alt="image" src="https://github.com/user-attachments/assets/e757b984-0ca0-4d08-be10-58c0cbc9e9af" />


---

## 🚀 使用方式

### 1. 安裝必要套件與啟動環境

在本地端執行 Jupyter Notebook 所需的 Python 套件：

```bash
pip install pandas requests openpyxl
```
### 2. 匯入並配置 n8n 工作流程
**匯入工作流**：進入 n8n 儀表板，點選右上角 「Import from File」，選取並匯入 自動化碳盤查排放量+人工審核 (1).json。 
**確認 Webhook 節點設定**：雙擊開啟最前端的 Webhook 節點，HTTP Method 為 POST，Path 為 /carbon/upload。 
**測試模式**：點擊節點內的 「Listen for Test Event」（測試 URL 為 http://localhost:5678/webhook-test/carbon/upload）。  
**正式模式**：將工作流切換為 Active（正式 URL 為 http://localhost:5678/webhook/carbon/upload）。
**掛載 Gmail 寄信憑證**：雙擊開啟末端的 碳盤查排放量審核資料_人工審核 (Gmail Node) 節點。  
在 Credential for Gmail OAuth2 欄位建立授權（登入 Gmail 帳號以啟用寄件權限）。  
於 sendTo 欄位填入審核人員收件信箱（預設範例：example@gmail.com）。

---

## 系統架構與資料流 (System Architecture)

```mermaid
flowchart TD
    subgraph Client ["1. 數據採集與批次傳送 (Python Client)"]
        A1[企業活動數據\nCSV / XLSX] --> A2[upload_data.py\n編碼轉換與清洗]
        A2 --> A3[封裝為 Multipart/form-data]
        A3 -->|HTTP POST 傳送 11 項 Binary 資料| B1
    end

    subgraph Engine ["2. 自動化工作流引擎 (n8n on Docker)"]
        B1[Webhook 接收端點\n/carbon/upload] --> B2[Extract from File\n解析二進位活動檔]
        B2 --> B3[Convert to JSON\n統一結構化陣列]
        B3 --> B4[Data Merge 整合閘道]
        B4 --> B5[計算核心 Code Node\nGHG Protocol / AR6 係數庫]
    end

    subgraph Audit ["3. 統計分析與報表封裝"]
        B5 --> C1[月度數據分流\nScope 1 / 2 / 3]
        B5 --> C2[Cat.1~9 排放熱點分析]
        C1 & C2 --> C3[P95 離群值檢定\n識別異常月份]
        C3 --> C4[生成 4 份核心稽核清冊]
        C4 --> C5[Compression Node\n封裝為 review_bundle.zip]
    end

    subgraph HITL ["4. 人機協作覆核 (Human-in-the-Loop)"]
        C5 --> D1[Gmail API 節點\n自動派發審核通知]
        D1 --> D2[永續部門專員 / 主管審查]
        D2 -->|確認無誤| D3[產出最終 ESG 盤查報告書]
        D2 -->|數據疑慮| D4[回饋修正原始活動數據]
    end
