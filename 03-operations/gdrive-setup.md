# Google Drive 整合設定指南

> 本文件為系統管理員操作手冊，說明如何將 Google Drive 資料夾作為 RAG 知識庫的資料來源。

---

## 目錄

1. [前置準備](#1-前置準備)
2. [系統設定](#2-系統設定)
3. [同步操作](#3-同步操作)
4. [CLI 操作](#4-cli-操作)
5. [常見問題排查](#5-常見問題排查)
6. [安全注意事項](#6-安全注意事項)

---

## 1. 前置準備

### 1.1 建立 Google Cloud Project

1. 前往 [Google Cloud Console](https://console.cloud.google.com/)
2. 點選上方專案選擇器，選擇「新增專案」
3. 輸入專案名稱（例如 `oda-cyber-gdrive`），點選「建立」
4. 等待專案建立完成後，確認已切換至該專案

### 1.2 啟用 Google Drive API

1. 在 Google Cloud Console 左側選單，前往「API 和服務」→「程式庫」
2. 搜尋 **Google Drive API**
3. 點選搜尋結果中的「Google Drive API」
4. 點選「啟用」按鈕
5. 等待 API 啟用完成（通常數秒內完成）

### 1.3 建立 Service Account 並下載金鑰

1. 在左側選單，前往「API 和服務」→「憑證」
2. 點選上方「建立憑證」→「服務帳戶」
3. 填寫服務帳戶資訊：
   - **服務帳戶名稱**：例如 `oda-gdrive-reader`
   - **服務帳戶 ID**：自動產生（可自行修改）
   - **說明**：`ODA Cyber Konsult Google Drive 讀取用途`
4. 點選「建立並繼續」
5. 角色設定可跳過（不需要授予 GCP 專案角色），直接點選「繼續」→「完成」
6. 在憑證頁面，點選剛建立的服務帳戶
7. 切換至「金鑰」分頁
8. 點選「新增金鑰」→「建立新金鑰」→ 選擇 **JSON** → 點選「建立」
9. 瀏覽器會自動下載一個 `.json` 金鑰檔案，請妥善保存

> **重要**：此 JSON 金鑰檔案包含私鑰，請勿提交至版本控制系統或外洩。

### 1.4 將 Service Account 加入 Google Drive 資料夾共用

1. 開啟下載的 JSON 金鑰檔案，找到 `client_email` 欄位，複製該 email 地址
   - 格式類似：`oda-gdrive-reader@oda-cyber-gdrive.iam.gserviceaccount.com`
2. 前往 Google Drive，找到要同步的目標資料夾
3. 在資料夾上點選右鍵 →「共用」→「與使用者和群組共用」
4. 貼上 Service Account 的 email 地址
5. 權限設定為「**檢視者**」（Viewer）即可
6. 取消勾選「通知使用者」（Service Account 無法接收通知）
7. 點選「共用」

> **提示**：記下資料夾的 Folder ID。在瀏覽器網址列中，Google Drive 資料夾的 URL 格式為：
> `https://drive.google.com/drive/folders/{FOLDER_ID}`
> 其中 `{FOLDER_ID}` 即為需要填入系統的資料夾 ID。

---

## 2. 系統設定

### 2.1 透過 Admin Dashboard 設定

系統提供圖形化介面進行 Google Drive 資料來源的設定，路由位於 `/datasources`。

**操作步驟：**

1. 登入 Admin Dashboard（`http://localhost:5173`）
2. 在左側側欄選擇「**資料來源**」（Data Sources）
3. 點選「新增資料來源」或選擇已有的 Google Drive 設定
4. 填寫以下資訊：
   - **資料夾 ID**（Folder ID）：貼上從 Google Drive 複製的資料夾 ID
   - **Service Account JSON**：點選「上傳」按鈕，選擇先前下載的 JSON 金鑰檔案
   - **同步排程**：選擇同步頻率（手動 / 每小時 / 每日 / 每週）
   - **標籤**（選填）：為此資料來源設定標籤，方便後續管理與篩選
5. 點選「**儲存**」

設定儲存後，系統會驗證 Service Account 的連線權限。若驗證成功，資料來源狀態將顯示為「已連線」。

### 2.2 環境變數說明

以下環境變數可在 `.env` 檔案中設定，用於調整 Google Drive 整合的行為：

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `RAG_GDRIVE_TEMP_DIR` | 檔案下載暫存目錄，同步完成後自動清理 | `data/gdrive_temp` |
| `RAG_GDRIVE_DEFAULT_TAG` | 從 Google Drive 同步的文件預設標籤 | `gdrive` |
| `RAG_GDRIVE_ENCRYPTION_KEY` | Fernet 對稱加密金鑰，用於加密儲存於資料庫中的 Service Account JSON | （必填，無預設值） |

**產生 Fernet 加密金鑰的方式：**

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

將產生的金鑰字串填入 `.env` 檔案：

```env
RAG_GDRIVE_ENCRYPTION_KEY=your-generated-fernet-key-here
```

> **注意**：`RAG_GDRIVE_ENCRYPTION_KEY` 為必填項目。若未設定，系統啟動時會拋出錯誤。此金鑰用於加密 Service Account JSON 的敏感資訊，請妥善保管並定期輪替。

---

## 3. 同步操作

### 3.1 手動觸發同步

**方式一：透過 Admin UI**

1. 前往 Admin Dashboard →「資料來源」頁面
2. 在對應的 Google Drive 資料來源卡片上，點選「**立即同步**」按鈕
3. 同步開始後，畫面會即時顯示同步進度

**方式二：透過 API**

```bash
curl -X POST http://localhost:4000/api/datasources/gdrive/sync \
  -H "Authorization: Bearer <your-jwt-token>" \
  -H "Content-Type: application/json" \
  -d '{"datasource_id": "<datasource-uuid>"}'
```

### 3.2 排程同步

系統支援以下四種同步排程：

| 排程模式 | 說明 |
|----------|------|
| `manual` | 僅手動觸發，不自動同步 |
| `hourly` | 每小時自動同步一次 |
| `daily` | 每日凌晨 02:00 自動同步 |
| `weekly` | 每週一凌晨 02:00 自動同步 |

排程設定於 Admin Dashboard 的資料來源設定中選擇，或透過 API 更新：

```bash
curl -X PATCH http://localhost:4000/api/datasources/gdrive/<datasource-uuid> \
  -H "Authorization: Bearer <your-jwt-token>" \
  -H "Content-Type: application/json" \
  -d '{"sync_schedule": "daily"}'
```

### 3.3 查看同步狀態與歷史

**透過 Admin UI：**

1. 前往「資料來源」頁面
2. 點選對應資料來源的「**同步歷史**」
3. 可查看每次同步的：
   - 開始時間與結束時間
   - 同步狀態（成功 / 失敗 / 進行中）
   - 新增、更新、刪除的檔案數量
   - 錯誤訊息（如有）

**透過 API：**

```bash
# 取得同步狀態
curl http://localhost:4000/api/datasources/gdrive/<datasource-uuid>/status \
  -H "Authorization: Bearer <your-jwt-token>"

# 取得同步歷史
curl http://localhost:4000/api/datasources/gdrive/<datasource-uuid>/history \
  -H "Authorization: Bearer <your-jwt-token>"
```

---

## 4. CLI 操作

系統提供 `oda-rag` CLI 工具，可在終端機中操作 Google Drive 同步功能，適用於開發除錯或伺服器端管理。

### 4.1 手動觸發同步

```bash
oda-rag gdrive sync
```

執行後會啟動同步流程，終端機即時顯示同步進度與結果摘要。

可搭配選項使用：

```bash
# 指定特定資料來源 ID
oda-rag gdrive sync --datasource-id <uuid>

# 強制全量同步（忽略 md5 比對，重新處理所有檔案）
oda-rag gdrive sync --force
```

### 4.2 顯示同步狀態

```bash
oda-rag gdrive status
```

輸出範例：

```
Google Drive 同步狀態
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
資料來源:      行銷部門文件
資料夾 ID:     1aBcDeFgHiJkLmNoPqRsT
排程:          daily
上次同步:      2026-02-11 02:00:15
狀態:          成功
檔案數:        42
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 4.3 列出已同步檔案

```bash
oda-rag gdrive files
```

輸出範例：

```
已同步檔案列表 (共 42 筆)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
檔案名稱                     類型    大小      同步時間
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
資安政策_v2.pdf              PDF     1.2 MB    2026-02-11 02:01
員工資安訓練手冊.docx         DOCX    856 KB    2026-02-11 02:01
風險評估表.xlsx              XLSX    234 KB    2026-02-11 02:02
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

可搭配篩選選項：

```bash
# 依檔案類型篩選
oda-rag gdrive files --type pdf

# 依標籤篩選
oda-rag gdrive files --tag gdrive
```

---

## 5. 常見問題排查

### 5.1 Service Account 無權限存取資料夾

**症狀：** 同步時出現 `403 Forbidden` 或 `The caller does not have permission` 錯誤。

**解決方式：**

1. 確認目標 Google Drive 資料夾已正確共用給 Service Account 的 email
2. 開啟 JSON 金鑰檔案，確認 `client_email` 欄位的 email 與共用的 email 一致
3. 確認共用權限至少為「檢視者」
4. 若為 Google Workspace 組織帳戶，確認組織管理員未限制外部共用

### 5.2 檔案格式不支援

**症狀：** 部分檔案同步後未出現在知識庫中。

**支援的檔案格式：**

| 格式 | 副檔名 | 解析方式 |
|------|--------|----------|
| PDF | `.pdf` | pdfminer.six |
| Word 文件 | `.docx` | python-docx |
| Excel 試算表 | `.xlsx` | openpyxl |
| CSV | `.csv` | pandas |
| 純文字 | `.txt` | 直接讀取 |
| Markdown | `.md` | 直接讀取 |
| HTML | `.html` | BeautifulSoup |
| JSON | `.json` | 直接讀取 |

不支援的檔案格式（如 `.pptx`、`.zip`、圖片檔案等）會在同步時被自動略過，並記錄於同步歷史中。

### 5.3 同步失敗

**症狀：** 同步狀態顯示「失敗」。

**排查步驟：**

1. 前往 Admin Dashboard →「資料來源」→「同步歷史」
2. 查看該次同步紀錄的 `error_message` 欄位
3. 常見錯誤原因：
   - **網路逾時**：Google Drive API 回應過慢，稍後重試即可
   - **配額超限**：Google Drive API 有每日呼叫次數限制，等待配額重置
   - **金鑰過期**：Service Account 金鑰可能被撤銷，需重新產生並上傳
   - **資料夾已刪除**：確認目標資料夾仍存在且未被移至垃圾桶

### 5.4 檔案修改後同步未更新

**症狀：** 在 Google Drive 上修改了檔案，但同步後知識庫內容未更新。

**說明：** 系統透過檔案的 **MD5 雜湊值**偵測變更。當檔案內容發生變化時，MD5 值會不同，系統會在下次同步時自動更新該檔案的向量索引。

**排查步驟：**

1. 確認修改已儲存至 Google Drive（非本機草稿）
2. 確認同步排程已觸發，或手動點選「立即同步」
3. 查看同步歷史中的「更新檔案數」是否大於 0
4. 若仍未更新，可嘗試使用 CLI 強制全量同步：
   ```bash
   oda-rag gdrive sync --force
   ```

---

## 6. 安全注意事項

### 6.1 Service Account JSON 的保護

- **加密儲存**：上傳的 Service Account JSON 金鑰在寫入資料庫前，會使用 Fernet 對稱加密進行加密。原始 JSON 內容不會以明文形式儲存
- **API 回傳隱藏**：透過 API 查詢資料來源設定時，回傳結果中不會包含 Service Account JSON 的完整內容，僅顯示 `client_email` 作為辨識用途
- **金鑰輪替**：建議定期（例如每季）輪替 `RAG_GDRIVE_ENCRYPTION_KEY`。輪替時需先以舊金鑰解密再以新金鑰重新加密

### 6.2 最小權限原則

- **專用 Service Account**：建議為本系統建立專用的 Service Account，不與其他系統共用
- **最小資料夾範圍**：僅將需要同步的特定資料夾共用給 Service Account，避免授予整個 Google Drive 的存取權限
- **僅檢視者權限**：Service Account 只需「檢視者」權限即可讀取檔案內容，無需「編輯者」或「管理者」權限
- **不授予 GCP 專案角色**：建立 Service Account 時不需要授予任何 GCP 專案層級的 IAM 角色

### 6.3 稽核與監控

- 所有同步操作皆記錄於系統的 `audit_logs` 資料表中
- 建議定期檢視同步歷史，確認無異常的存取行為
- 若發現未授權的存取跡象，應立即撤銷 Service Account 金鑰並重新產生

---

## 附錄：快速設定清單

以下為設定 Google Drive 整合的快速核對清單：

- [ ] 建立 Google Cloud Project
- [ ] 啟用 Google Drive API
- [ ] 建立 Service Account 並下載 JSON 金鑰
- [ ] 將 Service Account email 加入 Google Drive 資料夾共用（檢視者權限）
- [ ] 記下目標資料夾的 Folder ID
- [ ] 設定 `RAG_GDRIVE_ENCRYPTION_KEY` 環境變數
- [ ] 登入 Admin Dashboard，前往「資料來源」頁面
- [ ] 填入 Folder ID、上傳 SA JSON、選擇同步排程
- [ ] 儲存設定並確認狀態為「已連線」
- [ ] 執行首次手動同步，確認檔案正確匯入
