本工作流提供英文和中文 tags 兩個版本（LM Studio 用英文還是中文生成相片描述），請大家按需選擇。

## 🛠️ 事前準備 (Prerequisites)

如果是Ups-Ryzen 用戶，運行的環境已經爲你搭建好。請參考 Ups-Ryzen 使用說明的相關章節。
如果你是自己的機，你需要具備以下知識：
1. Docker容器基本安裝和使用
2. 映射Windows文件夾給Docker容器
3. 客製化Docker鏡像（因n8n原本鏡像並無 exif-tool 功能）
4. N8N的使用方法

在匯入此工作流之前，請確保您的環境符合以下條件：

1. **n8n Docker 容器**
2. **LM Studio**：
   * 已安裝並啟動 Local Server (預設通訊埠為 `1234`)。
   * 已載入支援視覺 (Vision) 的本地模型 (例如 `Qwen-3.5 4B`)。
   * 開啓 LM Studio 和 Windows Defender的網絡訪問，請確保能透過 `http://host.docker.internal:1234` 訪問到 LM Studio。
3. **資料夾結構**：
   請在 n8n 容器或伺服器中建立以下絕對路徑 (若使用 Docker，請將主機資料夾掛載至這些路徑)：
   * `/files/ai-image/inbox/` （你要處理的圖片請丟這裡）
   * `/files/ai-image/done/` （處理完成的圖片會出現在這裡）

---

## 🚀 安裝與使用指南 (Installation & Usage)

1. **複製工作流**：打開本專案中的 JSON 原始碼並複製。
2. **匯入 n8n**：在您的 n8n 畫面上，直接點擊鍵盤 `Ctrl + V` (或 `Cmd + V`) 貼上節點。
3. **確認exif-tool已經安裝**
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
