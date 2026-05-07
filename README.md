# n8n 本地 AI 圖片標籤自動化工作流 (嚴格循序版)
# Strict Sequential AI Image Tagging Workflow for n8n

這個 n8n 工作流旨在解決本地環境中監聽資料夾並呼叫 AI 模型處理圖片時，常見的**並發塞車 (Concurrency overload)** 與 **事件觸發無窮迴圈 (Infinite triggers)** 問題。

透過採用「排程拉取 (Spooler/Inbox-Done)」架構，此工作流能定期掃描收件資料夾，將圖片嚴格以「一圖接一圖」的順序送交本地 LM Studio 視覺模型 (如 Qwen VL 系列) 進行分析，並將 AI 生成的描述寫入圖片的 EXIF 資訊中。

## ✨ 主要特色

* **嚴格單執行緒處理 (Strict One-by-One Processing)**：使用 Loop 節點強制 Batch Size 為 1，確保本地 AI 模型不會因為同時湧入多張圖片而崩潰 (OOM)。
* **免疫無窮迴圈 (Infinite-Loop Immune)**：放棄傳統的 `Local File Trigger`，改用定時掃描。徹底解決「寫入新 EXIF -> 修改了檔案 -> 再次觸發流程」的死循環災難。
* **內建 Exif 節點 Bug 修復**：針對社群節點 `n8n-nodes-exif-data` 遺留 `.tmp` 暫存檔導致後續檔案處理失敗的已知 Bug，內建了自動清理機制作為 Workaround。
* **無縫檔案流轉**：自動讀取、標籤、搬移至完成區，並自動刪除原始檔案，保持收件夾整潔。

---

## 🛠️ 事前準備 (Prerequisites)

在匯入此工作流之前，請確保您的環境符合以下條件：

1. **n8n 實例**：建議透過 Docker 部署，確保有權限執行 Shell 指令 (`Execute Command` 節點)。
2. **LM Studio**：
   * 已安裝並啟動 Local Server (預設通訊埠為 `1234`)。
   * 已載入支援視覺 (Vision) 的本地模型 (例如 `Qwen-VL` 或 `Qwen2.5-VL`)。
   * 如果 n8n 運行在 Docker 中，請確保能透過 `http://host.docker.internal:1234` 訪問到 LM Studio。
3. **資料夾結構**：
   請在 n8n 容器或伺服器中建立以下絕對路徑 (若使用 Docker，請將主機資料夾掛載至這些路徑)：
   * `/files/ai-image/inbox/` （你要處理的圖片請丟這裡）
   * `/files/ai-image/done/` （處理完成的圖片會出現在這裡）

---

## 🚀 安裝與使用指南 (Installation & Usage)

1. **複製工作流**：打開本專案中的 JSON 原始碼並複製。
2. **匯入 n8n**：在您的 n8n 畫面上，直接點擊鍵盤 `Ctrl + V` (或 `Cmd + V`) 貼上節點。
3. **確認社群節點**：若系統提示缺少節點，請前往 n8n 的 Settings > Community Nodes 安裝 `n8n-nodes-exif-data`。
4. **檢查路徑與權限**：
   * 確認 `Execute Command` 節點中執行 `find` 和 `rm` 指令的權限與路徑是否與您的實際環境相符。
   * 確認 API URL (`http://host.docker.internal:1234/v1/chat/completions`) 在您的網路環境中是通的。
5. **啟動工作流**：將工作流切換為 **Active**。它將會預設每 1 分鐘醒來一次，檢查 `/inbox` 是否有新圖片。

---

## 🧩 工作流原理解析 (How it works)

本工作流包含以下關鍵步驟：

1. **Schedule Trigger**: 每 1 分鐘自動觸發一次。
2. **List Inbox Files**: 執行 Shell 指令 (`find /files/ai-image/inbox...`) 列出所有待處理檔案。若無檔案則直接結束，不消耗資源。
3. **Loop Over Items**: 核心節點！將陣列拆分成 Batch Size = 1，確保每次只釋出一張圖片進入後續流程。
4. **Convert to Base64**: 將 n8n 的二進位檔案指標 (filesystem-v2) 轉換回真實的 Base64 格式，以符合 LM Studio API 格式，同時防止 EXIF 節點解析錯誤。
5. **LM Studio Vision AI**: 發送 HTTP POST 請求至本地模型，要求以逗號分隔的格式 (限制 30 詞彙以內) 描述圖片內容 (人事時地物)。
6. **Clean Exif Temp File**: **[關鍵修復]** 執行 `rm -f` 強制刪除上次處理殘留的 `_exiftool_tmp` 檔案，避免後續寫入報錯。
7. **Write Exif Data**: 將 AI 的回覆擷取出來，寫入圖片的 `ImageDescription` 與 `Keywords` 標籤中。
8. **Save to Done / Delete Original**: 將完成標記的檔案存入 `/done` 資料夾，並將 `/inbox` 的原始檔案刪除，完成閉環。然後回到 Loop 節點處理下一張圖片。

---

## ⚠️ 注意事項與疑難排解 (Troubleshooting)

* **權限錯誤 (Permission Denied)？**
  確保 n8n 的執行身分 擁有 `/files/ai-image/` 目錄的完整讀寫 (RW) 權限。