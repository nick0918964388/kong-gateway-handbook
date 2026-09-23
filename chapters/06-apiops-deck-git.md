# 6 APIOps：decK 與 Git

正式環境不在 Kong Manager 手改設定。設定以 decK 宣告式檔案放在 Git，經 MR 審核、CI 驗證後由 `deck gateway sync` 套到各環境的 CP。

| 指令 | 做什麼 | 什麼時候用 |
|---|---|---|
| `deck gateway dump` | 把現有設定匯出成檔 | 初次納管、事故時比對漂移 |
| `deck gateway validate` / `deck file validate` | 檢查檔案格式與實體是否合法 | CI 第一步 |
| `deck gateway diff` | 顯示檔案與閘道的差異，不套用 | MR 審核、上線前 |
| `deck gateway sync` | 讓閘道等於檔案：**會刪除檔案裡沒有的實體** | 核簽後由 CI 執行 |
| `deck file openapi2kong` | OpenAPI 轉 deck 檔 | 以規格為單一真相 |

- **多團隊共用一個 CP**：一定用 `--select-tag` 分隔，否則 A 團隊 sync 會把 B 團隊的實體刪掉。
- **機密不進 Git**：金鑰、密碼用 `${{ env "DECK_XXX" }}` 由 CI 注入，或 Enterprise 的 Vault 參照。
- **環境差異**：一份基礎檔 + 每環境覆寫（`deck file patch` 或分目錄），不要複製四份。
