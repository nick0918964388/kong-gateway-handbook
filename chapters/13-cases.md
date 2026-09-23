# 13 實務常見問題與經驗案例

每個案例：症狀 → 原因 → 處置 → 預防。

### 案例 1：宣告式設定放了，Kong 卻完全沒有這些路由

**症狀**：ConfigMap 裡的 kong.yml 有 4 個 service，Admin API 查卻是空的。**原因**：容器同時設了 `KONG_DATABASE=postgres` 與 `KONG_DECLARATIVE_CONFIG`；DB 模式下宣告式檔會被忽略。**處置**：二選一——要 DB-less 就 `KONG_DATABASE=off`；要 DB 模式就用 deck sync 匯入。**預防**：部署清單加檢查：兩個變數不得同時存在。

### 案例 2：閘道 pod 反覆重啟上千次沒人發現

**症狀**：kong pod CrashLoopBackOff，重啟計數已達數千次，沒人察覺是因為還沒有正式流量。**原因**：單副本、沒有資源保留、沒有探針與告警。**處置**：補 requests/limits、readiness 用 /status/ready、PDB、副本數告警。**預防**：任何環境上線前先接監控，再接流量。

### 案例 3：升級到 3.x 後一堆 Route 變成 404

**症狀**：原本 `paths: ["/api/v1/.*"]` 的路由全部對不到。**原因**：3.0 新路由器規定正規式路徑必須以 `~` 開頭，否則當成字面前綴。**處置**：改成 `~/api/v1/.*`，或先用 `router_flavor: traditional_compatible` 過渡。**預防**：升級前用 deck validate + 路由測試集跑一遍。

### 案例 4：限流設 100 rps，實際放過了 300

**症狀**：壓測發現額度是設定的三倍。**原因**：rate-limiting `policy: local` 是每個 DP 各自計數，3 個 DP 就 3 倍。**處置**：改 `policy: redis`（或 Enterprise rate-limiting-advanced），限流計數集中。**預防**：文件寫清楚每個限流插件的 policy；壓測時看 `RateLimit-Remaining` 標頭。

### 案例 5：全域限流明明有掛，某條 Route 卻不受限

**症狀**：全域 rate-limiting 100/min，某 Route 另掛了 1000/min，結果全域的對它無效。**原因**：同名插件只執行優先權最高的一個實例，不合併。**處置**：把 Route 層的值設成想要的最終值。**預防**：規則寫進政策分層文件：全域是「預設」，範圍越小越優先且會完全取代。

### 案例 6：log 裡的 client_ip 全部是 Router 的 IP

**症狀**：所有請求 client_ip 都是 10.x（OCP Router／SLB），查不到真實來源。**原因**：沒設 `trusted_ips`，Kong 不信任 X-Forwarded-For。**處置**：`KONG_TRUSTED_IPS` 填 Router／SLB 網段、`KONG_REAL_IP_HEADER=X-Forwarded-For`。**預防**：入口拓樸一定畫出來，每一跳的來源 IP 都要有人負責傳遞。

### 案例 7：後端一台拿掉了，Kong 還是打過去 30 秒

**症狀**：下線一個 target 後仍有請求逾時。**原因**：只開了被動健康檢查，要等連續失敗 N 次才標 unhealthy；且 DNS 快取還指向舊 IP。**處置**：加主動健康檢查（10 秒間隔）；有計畫下線就先 `weight: 0` 再拿掉。**預防**：主動 + 被動一起開；維護前先降權重。

### 案例 8：503 failure to get a peer from the ring-balancer

**症狀**：所有請求 503，後端其實活著。**原因**：主動健康檢查路徑打錯（回 404）→ 全部 target 被標 unhealthy。**處置**：改對 http_path；緊急時暫時關 healthchecks。**預防**：健康檢查路徑要獨立、便宜、回 200，並在 SIT 驗證。

### 案例 9：deck sync 把別的團隊的 API 刪光

**症狀**：A 團隊 sync 自己的檔案後，B 團隊的 Route 全消失。**原因**：sync 的語意是「閘道 = 檔案」，檔案裡沒有的就刪，沒用 `--select-tag`。**處置**：從 Git 還原 B 的檔案 sync 回去（這就是 Git 的價值）。**預防**：共用 CP 一律 `--select-tag`；CI 先 diff 再 sync，diff 出現 delete 要人工核准。

### 案例 10：金鑰出現在 URL 和存取日誌裡

**症狀**：稽核在 log 看到 `?apikey=…`。**原因**：key-auth 的 `key_in_query` 預設允許。**處置**：關 `key_in_query`，只允許標頭；http-log 用 custom_fields_by_lua 移除 apikey／authorization；historical log 清洗。**預防**：安全基線文件列為必設項。

### 案例 11：回應轉換一開，大報表 API 記憶體暴增

**症狀**：DP 記憶體飆高、偶發 OOM，只在報表路由發生。**原因**：response-transformer 需要緩衝整個回應 body 才能改。**處置**：報表／匯出路由不掛轉換插件，另開路由；或在後端就遮罩。**預防**：遮罩只掛在標記 PII 且回應小的路由。

### 案例 12：DP 在 CP 停機後過一陣子重啟就起不來

**症狀**：CP 維護中，DP pod 被重新排程後一直 503。**原因**：DP 的設定快取在容器本地（emptyDir），pod 重建後快取沒了，CP 又不在。**處置**：恢復 CP。**預防**：CP 維護窗口不要動 DP；DP 快取可掛 PVC；CP 至少 2 副本。

### 案例 13：OIDC 登入後 cookie 太大被 Router 擋

**症狀**：使用者登入後 400 或 431。**原因**：openid-connect 預設把 session 存在 cookie，含 token 後超過 Router／瀏覽器上限。**處置**：session 改存 Redis（`session_storage: redis`），cookie 只留 session id。**預防**：OIDC 上線前確認 session 儲存策略。

### 案例 14：Admin API 開在 0.0.0.0，被掃到

**症狀**：資安掃描報告 8001 對外可達。**原因**：測試環境為方便設 `KONG_ADMIN_LISTEN=0.0.0.0:8001` 且 Service／Route 暴露。**處置**：Admin 只在 CP、只綁內網介面、NetworkPolicy 限來源、開 RBAC。**預防**：安全基線把 8001 列為禁止對外的埠，掃描納入 DAST。

### 案例 15：逾時設太長，後端一慢整個閘道連線用光

**症狀**：後端一支慢查詢拖到所有 API。**原因**：全域 read_timeout 300 秒，連線堆積把 worker 連線數吃滿。**處置**：慢查詢 API 獨立 Service 設長逾時，其餘維持 60 秒以下；加被動健康檢查斷路。**預防**：逾時屬工作區層預設，路由層只允許縮短或有理由地拉長。

### 案例 16 · Enterprise：授權到期，deck sync 全部失敗

**症狀**：CI 的 deck sync 回 4xx，Manager 顯示唯讀，流量正常。**原因**：企業授權到期後設定進入唯讀模式。**處置**：匯入新授權（`POST /licenses` 或更新 Secret 後重啟 CP）。**預防**：`kong_enterprise_license_expiry_timestamp` 進告警（60 天），續約流程納入年度行事曆。

### 案例 17 · Enterprise：兩個工作區各建了 /v1/orders，其中一個永遠對不到

**症狀**：B 工作區的 Route 建立成功，但流量都進 A 工作區的服務。**原因**：工作區隔離管理權限，不隔離路由；所有 Route 共用同一個路由器，同樣的 paths＋hosts 會由優先序決定或衝突。**處置**：以 hosts 或路徑前綴（`/aa/…`、`/bg/…`）區分系統。**預防**：政策分層文件規定每個工作區的路徑命名空間；CI 用 deck validate 加自訂檢查路徑重疊。

### 案例 18 · Enterprise：RBAC 管理 token 被寫進 Jenkinsfile

**症狀**：資安掃描在 Git 找到 `Kong-Admin-Token`。**原因**：為了讓 CI 跑 deck sync，直接把 super-admin token 貼進管線。**處置**：撤銷該 token；建 CI 專用 RBAC 使用者只給該工作區的寫入權；token 放 Vault／Secret 由 CI 注入。**預防**：gitleaks pre-receive 規則加 `Kong-Admin-Token`；每季盤點 RBAC 使用者。

### 案例 19 · Enterprise：rate-limiting-advanced 額度時準時不準

**症狀**：同一 consumer 有時 100 次就被擋，有時 130 次才擋。**原因**：`strategy: redis` 但 `sync_rate` 設太大（或 -1 表示只用本地計數），各 DP 之間同步延遲。**處置**：`sync_rate` 設 1 秒以內；對精確度要求高的路由用 `strategy: cluster`（DB 模式）或接受小誤差。**預防**：文件說明滑動視窗與同步頻率的取捨，壓測驗證。

### 案例 20 · Enterprise：Dev Portal 在 OpenShift 上打開是空白頁

**症狀**：portal 網址能開但內容不載，瀏覽器 console 一堆 CORS 錯誤。**原因**：`portal_gui_host`／`portal_api_url` 與實際 Route 網址不一致，或 portal API 沒開 Route。**處置**：兩條 Route（portal、portal-api）與設定值一致，TLS 一致。**預防**：部署清單把這兩個值當必填參數。

### 案例 21 · Enterprise：資料庫還原到新 CP 後，所有 consumer 的金鑰都失效

**症狀**：DR 演練還原 PostgreSQL，Manager 看得到 consumer，但 key-auth 全部 401。**原因**：開了 keyring 靜態加密，新 CP 沒有原本的金鑰，憑證欄位解不開。**處置**：匯入離線保管的 recovery key／keyring 匯出檔。**預防**：keyring 金鑰列入「金鑰與憑證」離線保管清單，演練步驟 1 一起核對。

### 案例 22 · Enterprise：Vault 維護時，OIDC 登入突然全部失敗

**症狀**：Vault 短暫停機半小時後，openid-connect 開始回 500。**原因**：client_secret 用 `{vault://hcv/…}` 參照，快取 TTL 到期後 DP 取不到值。**處置**：恢復 Vault；必要時暫時改為 `{vault://env/…}`。**預防**：Vault 做 HA；`resurrect_ttl` 設長於預期維護窗口；Vault 維護納入變更凍結。

{% hint style="success" %}
**經驗 · 排錯順序 — 五分鐘定位法**

1 看回應標頭 `X-Kong-Proxy-Latency` vs `X-Kong-Upstream-Latency`：誰慢。2 看狀態碼速查表：Kong 回的還是後端回的（http-log 的 `source` 欄位）。3 查 `/upstreams/{name}/health`：後端健不健康。4 看 prometheus 的連線數與 shared dict：閘道自己撐不住？5 `deck gateway diff`：設定有沒有漂移。多數問題在前三步就結束。
{% endhint %}

{% hint style="success" %}
**經驗 · 上線前檢核 — 十六項**

Admin API 不對外 ✓ trusted_ips 設好 ✓ 每個 Service 有逾時與 retries ✓ Upstream 主動+被動健康檢查 ✓ 限流 policy 非 local（多 DP）✓ key_in_query 關 ✓ http-log 移除機密標頭 ✓ prometheus 四個 flag 開 ✓ 探針用 /status/ready ✓ PDB 與反親和 ✓ 三種憑證到期告警 ✓ 每環境一組 CP，deck 用 select-tag ✓ 授權到期告警 ✓ RBAC 開且 CI 用專屬使用者 ✓ keyring／Vault 金鑰離線保管 ✓ audit_log 匯出日誌平台 ✓
{% endhint %}
