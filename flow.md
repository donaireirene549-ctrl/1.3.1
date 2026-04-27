# 1.3.1 確認訂單 — 系統流程圖（Mermaid 版）

> 反映目前程式現狀（含 7-11/FM 地址反查與降級、地址補齊、Sheets 模組化、Dashboard 複製錯誤等改動）

---

## 1. 系統架構

```mermaid
flowchart LR
    Browser[瀏覽器<br>Dashboard UI]
    Server[server.js<br>HTTP + SSE<br>:3001]
    Odoo[odoo.js<br>Playwright 爬蟲]
    Sheets[sheets.js<br>Google Sheets API]
    DB[(history.db<br>SQLite)]
    OAI[OpenAI API<br>gpt-4o / gpt-4o-mini]
    GS[(Google Sheets<br>7-11 / FM / Home)]
    OdooSrv[Odoo<br>lotri-inc.odoo.com]

    Browser <-->|HTTP / SSE| Server
    Server -->|spawn| Odoo
    Odoo -->|Playwright| OdooSrv
    Odoo -->|require| Sheets
    Odoo -->|recordOrder<br>finishSession| DB
    Odoo -->|askAI| OAI
    Sheets -->|values.update<br>batchUpdate| GS
    Server -->|read| DB
```

---

## 2. 主流程（odoo.js）

```mermaid
flowchart TD
    Start([npm start → 點 Dashboard 一鍵執行]) --> Login[登入 Odoo]
    Login --> Filter[套用「確認訂單用」篩選]
    Filter --> Scan[掃描所有列<br>取出非今日訂單清單]
    Scan --> EmptyCheck{有非今日<br>訂單?}
    EmptyCheck -->|無| End1([結束 — 沒事可做])
    EmptyCheck -->|有| Click[點第一筆非今日訂單]
    Click --> Loop[逐筆處理迴圈]
    Loop --> CheckToday{是今日<br>訂單?}
    CheckToday -->|是| BreakLoop[break 跳出迴圈]
    CheckToday -->|否| Process[處理單筆訂單<br>見圖 3]
    Process --> NextBtn[點下一筆 ›]
    NextBtn --> SeenBefore{已處理過?}
    SeenBefore -->|是| BreakLoop
    SeenBefore -->|否| Loop
    BreakLoop --> SaveJson[寫 order_result.json]
    SaveJson --> SaveDB[finishSession 寫 DB]
    SaveDB --> WriteSheets[呼叫 sheets.writeToGoogleSheetsAPI<br>見圖 4]
    WriteSheets --> Close([關閉瀏覽器])
```

---

## 3. 單筆訂單處理（核心邏輯）

```mermaid
flowchart TD
    Start([開始處理一筆訂單]) --> Step1{步驟1<br>Chatter 有訊息?}
    Step1 -->|無| Err1[❌ Message 無訊息]
    Step1 -->|有| Parse[parseMessage<br>解析總額 / 電話 / 客戶名 /<br>寄件方式 / 店號 / 地址 / 產品]
    Parse --> Step2{步驟2<br>訊息 NT.xxx<br>= 頁面總計?}
    Step2 -->|不一致| Err2[❌ 金額不一致]
    Step2 -->|一致| Step3{步驟3<br>訊息產品<br>= 訂單明細?}
    Step3 -->|不一致| Err3[❌ 產品不一致]
    Step3 -->|一致| Step4{步驟4<br>備註欄為空?}
    Step4 -->|有備註| Err4[❌ 備註不為空]
    Step4 -->|空| Ship[判斷寄件方式<br>見圖 5]

    Ship --> Confirm{有錯誤?}
    Confirm -->|是| LogErr[加入 errorLog<br>不點確認]
    Confirm -->|否| ClickConfirm[點「確認」按鈕<br>處理可能的彈窗]

    ClickConfirm --> Outlying{是外島?}
    Outlying -->|是| AddTag[切到「其他資訊」tab<br>加上「外島」標籤]
    Outlying -->|否| RecordDB[recordOrder 寫 DB]
    AddTag --> RecordDB
    LogErr --> RecordDB

    Err1 --> RecordDB
    Err2 --> RecordDB
    Err3 --> RecordDB
    Err4 --> RecordDB
    RecordDB --> End([結束此筆])
```

---

## 4. 寄件方式判斷與 7-11/FM → HOME 降級

```mermaid
flowchart TD
    Start([判斷寄件方式]) --> Type{shippingType?}

    Type -->|7-11 / FM| HasStore{訊息已有<br>店號?}
    Type -->|HOME| HasAddr{訊息已有<br>地址?}
    Type -->|無法判斷| ErrType[❌ 無法判斷寄件方式]

    HasStore -->|是| OK711[✅ 推進 shippingData.7-11/FM]
    HasStore -->|否| AIStore[AI 從圖片/截圖<br>辨識店號]
    AIStore --> StoreFound{找到店號?}
    StoreFound -->|是| OK711
    StoreFound -->|否| AIAddrCV[AI 從圖片/截圖<br>辨識地址]
    AIAddrCV --> AddrCVFound{找到地址?}
    AddrCVFound -->|是| AILookup[AI 反查附近<br>7-ELEVEN / 全家 門市]
    AddrCVFound -->|否| ErrStore[❌ 無法取得店號]
    AILookup --> LookupOK{回傳店號?}
    LookupOK -->|是| OK711
    LookupOK -->|否| Downgrade[降級 shippingType = HOME<br>標記 downgradedFrom]

    HasAddr -->|是| AddrComplete
    HasAddr -->|否| AIAddr[AI 從圖片/截圖<br>辨識地址]
    AIAddr --> AddrFound{找到地址?}
    AddrFound -->|是| AddrComplete
    AddrFound -->|否| ErrAddr[❌ 無法取得地址]

    Downgrade --> AddrComplete

    AddrComplete{地址含<br>縣/市 + 區/鄉/鎮 +<br>路/街/道 + 號?}
    AddrComplete -->|完整| GetPostal[askAI 查郵遞區號]
    AddrComplete -->|缺項| AICompleteAddr[AI 補齊地址<br>不確定就回 NONE]
    AICompleteAddr --> GetPostal
    GetPostal --> CheckIsland{地址含外島<br>地名?}
    CheckIsland -->|是| MarkIsland[標記 isOutlyingIsland]
    CheckIsland -->|否| OKHome
    MarkIsland --> OKHome[✅ 推進 shippingData.HOME]
```

---

## 5. Google Sheets 寫入（sheets.js）

```mermaid
flowchart TD
    Start([writeToGoogleSheetsAPI<br>shippingData]) --> LoadToken[讀 token.json]
    LoadToken --> TokenOK{token<br>存在?}
    TokenOK -->|否| WarnAuth[⚠ 提示跑 auth-google.js]
    TokenOK -->|是| InitOAuth[初始化 OAuth2 client<br>+ 自動 refresh 監聽]
    InitOAuth --> GetMeta[取得各 sheet 的 sheetId]
    GetMeta --> Loop711

    Loop711[7-11 寫入] --> Read711[讀 E 欄已存在<br>訂單編號]
    Read711 --> Filter711[過濾掉已存在的]
    Filter711 --> Has711{有新筆?}
    Has711 -->|否| LoopFM
    Has711 -->|是| Write711Header[第 2 列插入日期]
    Write711Header --> Write711Rows[每筆插入新列<br>電話用 ' 前綴強制文字]
    Write711Rows --> LoopFM

    LoopFM[FM 寫入] --> ReadFM[讀 F 欄已存在訂單編號]
    ReadFM --> WriteFMRows[同 7-11 流程]
    WriteFMRows --> LoopHome

    LoopHome[Home 寫入] --> ReadHome[讀 A 欄已存在訂單編號]
    ReadHome --> WriteHomeRows[同 7-11 流程<br>含郵遞區號 + 地址]
    WriteHomeRows --> Done([✅ 完成])

    InitOAuth -.invalid_grant.-> Catch[⚠ Token 過期<br>提示重跑 auth-google.js]
```

**注意事項：**
- 電話寫入時前綴 `'` → Sheets 視為純文字，不吃開頭的 0
- `+886`/`886` 開頭的會正規化成 `09********`
- 已存在的訂單編號會跳過，重跑安全
- 寫入日期 `writeDate` 在實際寫入時才取，跨午夜不會錯

---

## 6. Dashboard 控制流（server.js + dashboard.html）

```mermaid
flowchart LR
    User[使用者] -->|按一鍵執行| BtnRun[POST /api/run]
    BtnRun --> Spawn[spawn node odoo.js]
    Spawn --> Capture[捕獲 stdout/stderr]
    Capture --> Parse[parseLine 分類事件]
    Parse --> Broadcast[SSE broadcast 給所有客戶端]
    Broadcast --> UI[Dashboard UI 即時更新<br>進度條 / 統計 / 日誌 / 異常]

    User -->|按複製錯誤| Copy[copyErrors<br>navigator.clipboard]
    User -->|按停止| BtnStop[POST /api/stop<br>kill child process]
    User -->|按載入上次結果| Load[GET /api/last-result<br>讀 order_result.json]
    User -->|按歷史記錄| History[GET /history<br>讀 SQLite]
```

---

## 7. 三條失敗路徑摘要

| 情境 | 處理 | 標記 |
|---|---|---|
| 7-11/FM 訊息直接給店號 | 直接寫入 7-11/FM 工作表 | — |
| 7-11/FM 給圖片可辨識店號 | AI 從圖片讀店號 → 寫入 | — |
| 7-11/FM 只給地址 → 反查到門市 | AI 反查店號 → 寫入 | — |
| 7-11/FM 只給地址 → 反查不到 | **降級為 HOME** → 寫入 Home 工作表 | `downgradedFrom: '7-11'/'FM'` |
| HOME 地址不完整 | AI 補齊縣市區路號 → 查郵遞區號 → 寫入 | `addressCompleted: true` |
| 訊息/訂單不一致（金額/產品/備註）| 不點確認 → 寫到 errors | DB 中 `status='error'` |

---

## 8. Token 失效復原

```mermaid
flowchart LR
    Run[odoo.js 寫 Sheets] -->|invalid_grant| Fail[Sheets 寫入失敗<br>order_result.json 仍有資料]
    Fail --> ReAuth[node auth-google.js<br>瀏覽器完成授權]
    ReAuth --> NewToken[新 token.json]
    NewToken --> Retry[node -e require sheets +<br>讀 order_result.json 補寫]
    Retry --> Done([Sheets 補齊])
```

> 長期解：把 OAuth app 在 Google Cloud Console 從 Testing → In production，refresh token 就不會 7 天過期。
