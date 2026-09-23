# 15 學習路徑與練習

| 階段 | 目標 | 練習（可用本機三容器練習組：Kong DB-less + httpbin + log 接收器） |
|---|---|---|
| 第 1 天 觀念 | 讀完 1～4 章；能畫出請求路徑與插件順序 | 起 Kong DB-less + httpbin；建一個 Service、兩條 Route（前綴、正規式），用 curl 驗證 404／200 |
| 第 2 天 安全 | 7 章 | 加 key-auth + consumer + acl；驗證 401／403；關 key_in_query；設 hide_credentials |
| 第 3 天 韌性 | 8 章 | 建 Upstream 兩個 target + 主動／被動健康檢查；停掉一個 target 觀察摘除與恢復；設逾時讓 /delay/5 回 504 |
| 第 4 天 觀測 | 10 章 | 掛 prometheus 與 http-log；用 curl 打四種請求（200/401/500/慢），看 :8100/metrics 與接收器收到的 JSON；用 custom_fields_by_lua 加 correlation_id、刪 apikey |
| 第 5 天 APIOps | 6 章 | deck gateway dump → 改檔 → diff → sync；故意漏掉一條 Route 看 sync 的刪除行為；改用 --select-tag |
| 第 6 天 企業版 | 12 章 | 開 RBAC 與 Workspaces，建兩個工作區各一個唯讀角色；用 {vault://env/...} 參照金鑰；開 audit_log 看誰改了設定；rate-limiting-advanced 綁 consumer group 做分級限流 |
| 第 7 天 介接情境 | 14 章 | 建一條批次路由（獨立逾時、size limit、pre-function 擋沒帶 Idempotency-Key 的 POST）與一條出站路由（指向 httpbin，retries 0）；打幾筆看 http-log 兩邊都帶同一個 correlation_id，並用 custom_fields_by_lua 把一個欄位遮掉 |
| 進階 | 2、5、11 章 | 在 OCP 起 CP + DP（hybrid）；停 CP 看 DP 續跑；升級 CP 再升 DP；換叢集憑證 |

驗收：能獨立完成第 13 章「上線前檢核」十六項，並用「五分鐘定位法」處理一個模擬故障（例如把健康檢查路徑改錯）。
