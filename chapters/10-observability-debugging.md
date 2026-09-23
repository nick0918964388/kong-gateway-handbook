# 10 觀測與除錯

| 來源 | 拿什麼 | 怎麼開 |
|---|---|---|
| prometheus 插件（:8100/metrics） | 請求數／狀態碼、三段延遲直方圖、頻寬、後端健康、連線數、記憶體、CP 連線、憑證到期 | 全域掛載；`status_code_metrics、latency_metrics、bandwidth_metrics、upstream_health_metrics` 3.x 預設關要開；`per_consumer` 慎開 |
| http-log 插件 | 每筆請求 JSON：request／response／latencies／consumer／service／route／tries | 指向 Logstash 或收集器；`custom_fields_by_lua` 加欄位或刪機密標頭；queue 參數調批次與重試 |
| opentelemetry 插件 | trace（閘道 → 後端） | 指向 OTel Collector；取樣率 |
| 回應標頭 | `X-Kong-Proxy-Latency`（Kong 花的）、`X-Kong-Upstream-Latency`（後端花的）、`X-Kong-Request-Id` | 預設就有；現場排錯第一眼看這三個 |
| 錯誤日誌 | stderr（`KONG_PROXY_ERROR_LOG`） | `KONG_LOG_LEVEL=debug` 只在排錯時開 |

## 狀態碼速查（Kong 自己回的）

| 狀態碼 | 常見原因 | 先看哪裡 |
|---|---|---|
| 404 no Route matched | 路徑／host／method 沒對到任何 Route | Route 的 paths 是否少了 `~`、hosts 有沒有設、strip_path |
| 401 / 403 | 認證插件擋（沒帶、錯、過期）／acl 拒絕 | 回應 body 的 message；consumer 有無綁憑證與群組 |
| 426 | Route 只允許 https，請求是 http | Route protocols |
| 429 | 限流 | policy 是 local 還是 redis；`RateLimit-*` 標頭 |
| 499 | 客戶端先斷線 | 客戶端逾時比 Kong 短 |
| 502 Bad Gateway | 後端拒絕連線或回應格式錯 | Target 位址、port、DNS；後端 log |
| 503 failure to get a peer from the ring-balancer | Upstream 所有 target 都 unhealthy 或沒有 target | 健康檢查狀態（Admin API `/upstreams/{name}/health`） |
| 504 | 後端超過 read_timeout | `X-Kong-Upstream-Latency`、後端慢查詢 |
| 500 + `lua_shared_dict` 錯誤 | 共享記憶體用光（限流計數、快取） | `kong_memory_lua_shared_dict_bytes`；調大 `mem_cache_size` 或對應 dict |
