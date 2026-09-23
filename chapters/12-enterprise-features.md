# 12 企業版功能全覽

企業版 = 同一個閘道核心 + 一組「治理與合規」功能。本章依「導入時會碰到的順序」整理：授權、管理面與權限、開發者入口、機密與加密、稽核、企業插件、消費者分群、合規模式、混合模式差異、支援與升級。

## 12.1 授權（License）與到期行為

- 授權是一個 JSON 檔，放在 Secret 掛給 CP（hybrid 模式由 CP 推給 DP）或以 Admin API `POST /licenses` 匯入；Helm 用 `enterprise.license_secret`。
- **到期後**：流量照常代理、既有設定繼續用，但所有設定變成**唯讀**——Admin API、Manager、deck sync 都不能改，直到匯入新授權。
- 預警：Kong 日誌在到期前 90 天與 30 天各警示一次，Manager 在 15 天前提示；指標 `kong_enterprise_license_expiry_timestamp` 要進告警（建議 60 天）。
- 沒有授權的「free mode」已宣告淘汰，未來等同到期行為；不要拿免授權的 Enterprise 映像當測試環境，測試環境也要有授權。

## 12.2 Kong Manager、Workspaces、RBAC

- **Workspace**：實體的命名空間，一個系統或團隊一個；`default` 工作區永遠存在。注意：**所有工作區的 Route 共用同一個路由器**，工作區隔離的是管理權限，不是路徑——兩個工作區宣告同樣的 paths 會互相搶或被拒。
- **RBAC**：`enforce_rbac=on` 後 Admin API 與 Manager 都要 `Kong-Admin-Token`。內建角色：super-admin（跨工作區）、admin、read-only、portal-admin（每工作區）；可自訂角色到「哪個工作區、哪類端點、哪些動作」。
- **管理者登入**：`admin_gui_auth` 支援 basic-auth、openid-connect（接同一個 IdP，最建議）、ldap-auth-advanced；`admin_gui_session_conf` 設 session 秘密與逾時。
- **實務**：Manager 給人「看」，改設定走 decK；CI 用專屬 RBAC 使用者與 token，token 放 Secret 或 Vault，不進 Git；每季盤點 RBAC 使用者。

```text
# kong.conf（CP）
enforce_rbac = on
admin_gui_auth = openid-connect
admin_gui_auth_conf = {"issuer":"https://idp.example/realms/main","client_id":"kong-manager","client_secret":"{vault://env/MANAGER_OIDC_SECRET}","admin_claim":"preferred_username","authenticated_groups_claim":["groups"]}
admin_gui_session_conf = {"secret":"{vault://env/SESSION_SECRET}","cookie_secure":true,"rolling_timeout":3600}
audit_log = on
```

## 12.3 Dev Portal（開發者入口）

- 每個工作區可開一個入口：API 目錄與文件（直接餵 OpenAPI）、開發者註冊與審核、**應用程式註冊**（開發者自己建 app，系統核發 key 或綁 OIDC client，對應到 Kong 的 consumer 與憑證）。
- 設定：`portal=on`、`portal_gui_host`、`portal_api_url`、`portal_auth`（basic／key／openid-connect）。在 OpenShift 上 portal 與 portal API 各需一條 Route，網址要跟這兩個設定一致，否則空白頁或 CORS 錯誤。
- 價值：把「申請 API 金鑰」從人工單據變成自助且留痕的流程，直接回應 RFP 的可追溯性。

## 12.4 Secrets Management（機密不落地）

- 任何設定值可用參照取代明文：`{vault://hcv/kong/redis/password}`（HashiCorp Vault）、`{vault://aws/…}`、`{vault://gcp/…}`、`{vault://azure/…}`、`{vault://env/VAR}`（環境變數，OSS 也有）。
- 可用在插件設定（例如 openid-connect 的 client_secret、rate-limiting-advanced 的 redis 密碼）與 kong.conf。
- 運作：DP 在需要時向 Vault 取值並依 TTL 快取；所以 **DP 也要能連到 Vault**，網路規則要開；Vault 不可用時用快取，快取過期後該插件失敗——Vault 要 HA。
- 輪替：改 Vault 裡的值，Kong 依 `ttl`／`resurrect_ttl` 自動更新，不用重佈設定。

## 12.5 Keyring（資料庫靜態加密）

- `keyring_enabled=on` 後，consumer 憑證等敏感欄位寫進 PostgreSQL 前先加密；金鑰由 Kong 產生或由 Vault 管理。
- **一定要匯出並離線保管 recovery key**：資料庫還原到新 CP 時沒有金鑰，所有憑證都解不開——跟 Manage 的 encryption secret 是同一類生命線。

## 12.6 Audit Log（管理稽核）

- `audit_log=on` 後記錄兩類：Admin API 每個請求（誰、何時、哪個端點、結果）與實體變更（改了哪個物件、前後值）；開 RBAC 時會帶 `rbac_user_name`。
- 存在 PostgreSQL，可用 `audit_log_ignore_methods / _paths / _tables` 降噪；建議定期匯出到日誌平台／SOC 並設保存期，這是「誰改了閘道設定」的唯一權威紀錄。

## 12.7 企業插件重點

| 插件 | 做什麼 | 導入時要決定的事 |
|---|---|---|
| **openid-connect** | 對接 OIDC IdP：授權碼流程（瀏覽器）、client credentials（系統對系統）、bearer token 驗證與 introspection；可把 claims 轉成標頭給後端 | session 存哪（cookie 會過大 → Redis）；哪些 claims 對應 consumer 與群組；token 快取 TTL；登出流程 |
| **mtls-auth** | 用客戶端憑證認證，對應 CA 與 consumer；支援憑證撤銷檢查 | CA 誰發、憑證輪替、Router 是否 passthrough 給 DP |
| **rate-limiting-advanced** | 多視窗（秒／分／時）、滑動視窗、Redis Sentinel／Cluster 集中計數、`sync_rate` 控制本地與 Redis 同步頻率；可依 consumer group 覆寫額度 | strategy 用 redis；identifier 依 consumer 還是 header；各群組額度表 |
| **proxy-cache-advanced** | Redis 後端的回應快取、依標頭／狀態碼條件、bypass 規則 | 哪些查詢類 API 可快取、TTL、快取鍵含不含身分 |
| **request-validator** | 依 JSON Schema／OpenAPI 參數驗證請求，不合格在閘道就擋 | 規格檔為單一真相；錯誤格式 |
| **response-transformer-advanced / jq / exit-transformer** | 條件式回應改寫、jq 表達式欄位級遮罩、統一錯誤回應 | PII 字典、遮罩掛哪些路由（避免大回應） |
| **opa** | 把授權決策交給 Open Policy Agent | 複雜授權規則才用 |
| **kafka-log / kafka-upstream / statsd-advanced** | 把 log 或請求送 Kafka；指標推 StatsD | 日誌平台若收 Kafka 可用 |
| **vault-auth** | 以 Vault 管理的憑證做認證 | 與 12.4 搭配 |
| **ai-proxy / ai-prompt-guard / ai-sanitizer** | AI Gateway：代理 LLM、提示詞守門、PII 清理（給 LLM 流量） | 一般 API 專案暫不需要，知道有即可 |

```yaml
# 分級限流：一般群組 100/分、財務系統 1000/分（3.4+ 插件可綁 consumer group）
consumer_groups:
- name: general
- name: finance-systems
plugins:
- name: rate-limiting-advanced
  consumer_group: general
  config: { limit: [100], window_size: [60], strategy: redis, sync_rate: 1,
            redis: { host: "{vault://env/REDIS_HOST}", password: "{vault://env/REDIS_PASSWORD}" } }
- name: rate-limiting-advanced
  consumer_group: finance-systems
  config: { limit: [1000], window_size: [60], strategy: redis, sync_rate: 1,
            redis: { host: "{vault://env/REDIS_HOST}", password: "{vault://env/REDIS_PASSWORD}" } }
```

## 12.8 Consumer Groups（消費者分群）

- 把 consumer 分成群組（一般、財務系統、稽核…），3.4 起**插件可以直接綁群組**：同一條路由掛多組插件實例，各綁一個群組，達成分級限流、分級遮罩、分級快取。
- 群組來源可以是靜態指定，也可由 openid-connect 依 token 的群組 claim 動態對應。

## 12.9 合規：FIPS 模式

- 提供 FIPS 140-2 版映像（`kong/kong-gateway-fips`），`fips=on` 後只用合規的加密演算法；政府案若資安規範要求可採用，效能略降、部分舊演算法不可用。

## 12.10 企業版在 Hybrid 模式的差異

- 授權只放 CP，自動推給 DP；DP 啟動時若拿不到授權，企業插件不生效。
- DP 對 CP 的遙測通道（:8006）原供 Vitals 使用，3.5 起 Vitals 移除，用量分析改由 Prometheus 插件與 log。
- Konnect（SaaS 控制面）與自建 Enterprise 功能大致相同，差在控制面誰維運、設定與指標資料存哪裡；政府或金融類專案基於資料留置多採自建。

## 12.11 支援與升級

- 支援等級：Business（營業時間）、Platinum（24x7、Sev-1 一小時）、Diamond（24x7、30 分鐘）；合約要寫明等級與升級路徑。
- 企業版映像 `kong/kong-gateway:3.x.y.z`（四段版號）；升級一律先 CP（`kong migrations up` → `finish`）再 DP；只跟 LTS。
