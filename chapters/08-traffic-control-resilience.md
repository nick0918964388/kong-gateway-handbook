# 8 流量控制與韌性

| 機制 | 設定在哪 | 重點 |
|---|---|---|
| 逾時 | Service：`connect_timeout / read_timeout / write_timeout`（預設 60000 ms） | 長查詢 API 獨立 Service 設較長；不要全域拉長 |
| 重試 | Service：`retries`（預設 5） | 只在連線失敗／逾時時換下一台；POST 這類非冪等請求「已送出後逾時」預設不重送。交易類 API 建議明確設 0 或 1，並讓後端支援冪等鍵 |
| 主動健康檢查 | Upstream：`healthchecks.active`（路徑、間隔、成功／失敗次數） | 路徑要便宜、要存在；預設關閉 |
| 被動健康檢查 | Upstream：`healthchecks.passive`（逾時、5xx 連續次數） | 只能標 unhealthy，不能自動恢復 → 一定搭主動檢查 |
| 負載平衡 | Upstream：`algorithm` round-robin / least-connections / consistent-hashing | 一致性雜湊要指定 hash_on（consumer、header…） |
| 限流 | rate-limiting（policy local / cluster / redis） | `local` 是每個 DP 各算：3 個 DP 等於 3 倍額度；跨 DP 一致要 redis（或 Enterprise 的 rate-limiting-advanced） |
| 降級 | proxy-cache（查詢類）、request-termination（維護模式）、Enterprise exit-transformer（統一錯誤格式） | proxy-cache 只快取 GET 且需 cache key 設計 |

```yaml
upstreams:
- name: maximo-api
  algorithm: least-connections
  healthchecks:
    active:
      type: http
      http_path: /maximo/api/ping     # 不需登入、回 200；/maximo/oslc/* 未登入會 302 到 IdP
      healthy:   { interval: 10, successes: 2 }
      unhealthy: { interval: 10, http_failures: 3, timeouts: 3 }
    passive:
      unhealthy: { http_failures: 3, timeouts: 3, http_statuses: [500, 502, 503, 504] }
  targets:
  - target: maximo-api-a.mas.svc:9080
  - target: maximo-api-b.mas.svc:9080
services:
- name: maximo-api
  host: maximo-api          # 指向 upstream 名稱
  connect_timeout: 5000
  read_timeout: 60000
  write_timeout: 60000
  retries: 1
```
