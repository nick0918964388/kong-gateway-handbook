# 9 轉換與遮罩

- **request-transformer**：加／改／刪請求標頭、query、body 欄位（例如加系統代碼、移除客戶端傳來的內部標頭）。
- **response-transformer**：對 JSON 回應加／改／刪欄位；OSS 版是靜態規則。
- **response-transformer-advanced / jq** `Enterprise`：可依狀態碼條件、可用 jq 表達式做欄位級改寫，遮罩（保留首尾、打星號）靠這個。
- **依身分動態遮罩**：同一條 Route 掛多組轉換插件實例，各綁不同 consumer group；或自訂插件讀 token claims。Kong 沒有現成的「Data Masking」開關。
- **代價**：body 轉換會關掉串流、把整個回應讀進記憶體再改；大回應（上千筆清單、檔案下載）不要掛，或另開不遮罩的路由給批次。
