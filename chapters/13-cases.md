# 13 實務常見問題與經驗案例

每個案例四段：**情境**（發生了什麼，包含管理面為什麼會走到這一步）→ **影響**（對系統、對業務、對團隊）→ **解法**，分成兩種：

- `標準` Kong 內建設定或官方插件（OSS 或 Enterprise）就能解決，不用寫程式。
- `客製` 要自訂插件（Lua／PDK）、轉接服務，或是流程與制度（申請單、凍結窗口、盤點、責任歸屬）。多數案例真正的根因在這一層。

案例分三組：13.1 技術與設定、13.2 企業版功能、13.3 管理與流程。最後是排錯順序與上線前檢核。

## 13.1 技術與設定

### 案例 1 · 技術：宣告式設定放了，Kong 卻完全沒有這些路由

**情境**：團隊把 kong.yml 放進 ConfigMap，又照另一份範例設了 `KONG_DATABASE=postgres`。上線前才發現 Admin API 查出來是空的；更麻煩的是 ConfigMap 有一份、資料庫裡是舊的一份，沒人說得清「應該以哪份為準」。

**影響**：兩份設定各說各話，任何人改了都不確定有沒有生效；上線當天路由全部 404。

`標準` DB-less 就 `KONG_DATABASE=off` + `KONG_DECLARATIVE_CONFIG`；DB 模式就用 deck sync 匯入。兩個變數不能同時存在。

`客製` 部署管線加一條檢查（兩變數互斥即失敗）；團隊公約：正式環境只有一個真相來源，就是 Git 裡的 deck 檔。

### 案例 2 · 技術：閘道 pod 反覆重啟上千次沒人發現

**情境**：單副本、沒有 requests/limits、沒有探針，重啟數也沒接進告警。測試期間沒流量，所以看起來「沒壞」；第一次壓測才發現重啟計數已經數千次。

**影響**：同樣的設定到正式環境，尖峰時會反覆重啟，每次都掉連線；而且沒人會知道，直到使用者投訴。

`標準` requests/limits、readiness 用 `/status/ready`、PDB、副本 ≥3；kube-state-metrics 的重啟數進告警。

`客製` 上線檢核表寫入「先接監控，再接流量」；重啟告警的接收人指定到值班群組，不是某個人的信箱。

### 案例 3 · 技術：升級到 3.x 後一堆 Route 變成 404

**情境**：升級由平台團隊執行，路由是各系統團隊寫的；沒人讀 breaking changes，也沒有路由測試集。升級後所有正規式路徑（`/api/v1/.*`）全部對不到。

**影響**：多個系統同時中斷；回滾要協調三個團隊，耗掉一整個晚上。

`標準` 3.0 起正規式路徑要以 `~` 開頭；過渡期用 `router_flavor: traditional_compatible`；升級前 `deck validate`。

`客製` 建「路由測試集」（每條 Route 一個 curl 案例，CI 跑）；升級流程加各系統團隊簽核與回滾點。

### 案例 4 · 技術：限流設 100 rps，實際放過了 300

**情境**：規格書寫 100 rps，壓測到 300 才被擋。`policy: local` 是每個 DP 各自計數，3 個 DP 就是 3 倍。後端團隊當初是以「閘道會擋在 100」為前提設計容量的。

**影響**：後端在尖峰承受三倍流量；規格與實際不一致，稽核時說不清楚。

`標準` `policy: redis`（OSS）或 rate-limiting-advanced 的 `strategy: redis`（Enterprise），計數集中。

`客製` 介接規格明訂「額度是全域還是每副本」；壓測報告附 `RateLimit-Remaining` 標頭為證。

### 案例 5 · 技術：全域限流明明有掛，某條 Route 卻不受限

**情境**：平台團隊在全域掛 100/分當底線，系統團隊在自己的 Route 掛 1000/分，以為兩個都會生效、取嚴者。實際只有 Route 那個生效。

**影響**：「底線」失效而平台以為有保護；出事時才發現保護從來沒有作用。

`標準` 同名插件只執行優先權最高的一個實例（Consumer+Route+Service → … → 全域），不合併；Route 層要寫最終值。

`客製` 政策分層文件：全域是預設、越小範圍越優先且完全取代；CI 加檢查「Route 層限流不得高於工作區上限」。

### 案例 6 · 技術：log 裡的 client_ip 全部是 Router 的 IP

**情境**：資安要查一個異常呼叫的來源，http-log 裡全是 10.x 的 Router IP。入口拓樸沒人畫，X-Forwarded-For 由誰負責傳遞沒人講。

**影響**：稽核與資安調查做不到來源追溯，等於沒有留痕。

`標準` `KONG_TRUSTED_IPS` 填 Router／SLB 網段、`KONG_REAL_IP_HEADER=X-Forwarded-For`。

`客製` 入口拓樸圖每一跳標「來源 IP 由誰傳遞」；上線檢核加一條實測「log 的 client_ip 是真實 IP」。

### 案例 7 · 技術：後端一台拿掉了，Kong 還是打過去 30 秒

**情境**：後端團隊維護時直接下線一台，沒通知閘道團隊。閘道只開被動健康檢查，要連續失敗才摘除；DNS 快取又還指向舊 IP。

**影響**：維護窗口內每 N 次請求就有一次逾時，使用者以為系統不穩。

`標準` 主動 + 被動健康檢查一起開；計畫下線先 `weight: 0` 再移除；`dns_stale_ttl` 調短。

`客製` 後端維護 SOP 加「先降權重、確認 `/upstreams/{name}/health` 顯示 unhealthy 再拆」，維護單會簽閘道團隊。

### 案例 8 · 技術：503 failure to get a peer from the ring-balancer

**情境**：主動健康檢查路徑打錯（回 404），Kong 把所有 target 標成 unhealthy；後端其實好好的。值班人員看到 503 先去重啟後端，越弄越亂。

**影響**：整條 API 全斷，還誤導了排錯方向。

`標準` `http_path` 指向獨立、便宜、回 200 的端點（例如 Manage 的 `/maximo/api/ping`）；緊急時暫時關 healthchecks。

`客製` 排錯手冊第一步加「503 先查 `/upstreams/{name}/health`」；健康檢查路徑列入介接規格，由後端保證存在。

### 案例 9 · 技術：deck sync 把別的團隊的 API 刪光

**情境**：多團隊共用一個 CP，A 團隊的 CI 跑 deck sync 沒帶 `--select-tag`，B 團隊的 Route 全消失。B 團隊第一時間不知道是誰刪的，也沒有變更紀錄可查。

**影響**：跨團隊事故；沒有紀錄就沒有責任歸屬，只剩互相猜測。

`標準` 一律 `--select-tag`；CI 先 diff，出現 delete 要人工核准；Enterprise 用 Workspaces + RBAC 從權限上隔開。

`客製` Git 是還原來源（從 B 的檔案 sync 回去）；制度：每團隊一個目錄 + CODEOWNERS，跨目錄變更要對方 approve。

### 案例 10 · 技術：金鑰出現在 URL 和存取日誌裡

**情境**：某系統為了方便把 apikey 放在 query string；`key_in_query` 預設允許，於是金鑰進了 Router、Kong、日誌平台每一層的 log。資安稽核抓到。

**影響**：金鑰形同外洩，要全面輪替；歷史 log 要清洗；稽核報告留紀錄。

`標準` `key_in_query: false`、只允許標頭；`hide_credentials`；http-log 的 `custom_fields_by_lua` 移除 apikey／authorization。

`客製` 安全基線文件列為必設項；金鑰輪替流程（同一 consumer 先加新、再刪舊）；日誌清洗腳本。

### 案例 11 · 技術：回應轉換一開，大報表 API 記憶體暴增

**情境**：資安要求所有回應遮罩身分證，團隊把 response-transformer 掛全域。報表 API 一次回幾萬筆，body 全部緩衝進記憶體，DP OOM。

**影響**：遮罩需求把不相干的報表 API 拖垮；DP 重啟波及所有 API。

`標準` 遮罩只掛在標記 PII 且回應小的路由；報表／匯出另開路由，不掛任何 body 轉換。

`客製` PII 欄位清單由業務單位維護（哪些 API 有哪些欄位）；大回應的遮罩改在後端或報表層做。

### 案例 12 · 技術：DP 在 CP 停機後過一陣子重啟就起不來

**情境**：CP 做維護，平台團隊同一時段做節點排程，DP pod 被重建；快取在 emptyDir，重建後沒有設定，CP 又不在。

**影響**：「CP 停機不影響流量」的保證失效；兩個團隊的維護窗口撞在一起，事後才發現沒人協調。

`標準` CP ≥2 副本；DP 快取掛 PVC；CP 維護窗口不動 DP。

`客製` 變更行事曆：平台維護與閘道維護互斥；CP 維護前確認 DP 所在節點沒有排程中的作業。

### 案例 13 · 技術：OIDC 登入後 cookie 太大被 Router 擋

**情境**：openid-connect 預設把 session 存 cookie，含 token 後超過 Router／瀏覽器上限。只有群組多、claims 多的使用者會遇到，QA 用一般帳號測不出來。

**影響**：特定使用者 400／431 登不進去，客服找不到規律，問題拖了兩週。

`標準` `session_storage: redis`，cookie 只留 session id。

`客製` OIDC 上線前用「群組最多的帳號」當測試案例；claims 精簡由 IdP 團隊配合。

### 案例 14 · 技術：Admin API 開在 0.0.0.0，被掃到

**情境**：測試環境為方便設 `KONG_ADMIN_LISTEN=0.0.0.0:8001` 並開了 Route，之後這份設定被整份複製到正式。資安掃描報告出來才發現。

**影響**：Admin API 對外等於整個閘道可被改寫；要走資安事件通報。

`標準` Admin 只在 CP、只綁內網介面、NetworkPolicy 限來源、開 RBAC（Enterprise）。

`客製` 安全基線把 8001 列為禁止對外埠；DAST 掃描納入上線關卡；環境設定不可整份複製，用每環境覆寫檔。

### 案例 15 · 技術：逾時設太長，後端一慢整個閘道連線用光

**情境**：一支報表 API 要跑 4 分鐘，團隊把全域 read_timeout 拉到 300 秒。後端一慢，連線堆積把 worker 連線吃滿，其他 API 一起變慢。

**影響**：一支慢 API 拖垮全部；閘道看起來像「當了」。

`標準` 慢 API 獨立 Service 設長逾時，其餘維持 60 秒以下；被動健康檢查斷路。

`客製` 逾時屬工作區層預設，Route 只能縮短，拉長要理由與審核；慢 API 改成非同步（202 + 查詢）。

## 13.2 企業版功能 `Enterprise`

### 案例 16 · Enterprise：授權到期，deck sync 全部失敗

**情境**：採購時授權是一年期，續約沒有指定負責人。到期那天 CI 的 deck sync 開始回 4xx、Manager 變唯讀；流量正常所以沒人急，直到要緊急改設定。

**影響**：緊急變更做不了；續約要走採購流程，最快也要幾天。

`標準` `kong_enterprise_license_expiry_timestamp` 進告警（60 天）；匯入新授權 `POST /licenses` 或更新 Secret 後重啟 CP。

`客製` 續約列入年度行事曆與採購時程（提前 90 天）；授權檔放 Secret 並納入備份清單與交接文件。

### 案例 17 · Enterprise：兩個工作區各建了 /v1/orders，其中一個永遠對不到

**情境**：工作區給各系統團隊自治權，大家以為工作區像命名空間一樣隔離路徑。B 工作區的 Route 建立成功，但流量都進 A 工作區的服務。

**影響**：跨系統資料誤送；問題只在特定路徑出現，定位花了很久。

`標準` 工作區隔離的是管理權限，不是路由；以 hosts 或路徑前綴（`/aa/…`、`/bg/…`）區分系統。

`客製` 政策文件規定每個工作區的路徑命名空間；CI 用 deck validate 加自訂檢查偵測路徑重疊。

### 案例 18 · Enterprise：RBAC 管理 token 被寫進 Jenkinsfile

**情境**：CI 要跑 deck sync，工程師直接把 super-admin 的 `Kong-Admin-Token` 貼進管線；資安在 Git 掃到。

**影響**：任何能讀 repo 的人都能改整個閘道；token 要撤銷，還要盤點誰用過。

`標準` CI 專用 RBAC 使用者只給該工作區寫入權；token 放 Vault／Secret 由 CI 注入。

`客製` gitleaks pre-receive 規則加 `Kong-Admin-Token`；每季盤點 RBAC 使用者與 token。

### 案例 19 · Enterprise：rate-limiting-advanced 額度時準時不準

**情境**：財務系統抱怨「同樣 100 次，有時 100 就擋、有時 130 才擋」。`strategy: redis` 但 `sync_rate` 設太大，各 DP 之間同步有延遲。

**影響**：對方無法設計重試邏輯；投訴閘道「不穩」，信任受損。

`標準` `sync_rate` 設 1 秒以內；要精確用 `strategy: cluster`（DB 模式）；文件說明滑動視窗的誤差。

`客製` 介接規格寫明「額度為近似值」與建議退避方式；壓測報告附證據。

### 案例 20 · Enterprise：Dev Portal 在 OpenShift 上打開是空白頁

**情境**：`portal_gui_host`／`portal_api_url` 與實際 Route 網址不一致，平台團隊建 Route 時用了不同網域；瀏覽器 console 一堆 CORS 錯誤。

**影響**：開發者入口上線延期；自助申請流程退回人工。

`標準` 兩條 Route（portal、portal-api）與設定值一致，TLS 一致。

`客製` 部署清單把這兩個值當必填參數；驗收加「用外部瀏覽器開 portal」。

### 案例 21 · Enterprise：資料庫還原到新 CP 後，所有 consumer 的金鑰都失效

**情境**：DR 演練還原 PostgreSQL，Manager 看得到 consumer，但 key-auth 全部 401。開了 keyring 靜態加密，recovery key 沒有離線保管，新 CP 解不開憑證欄位。

**影響**：DR 演練失敗，所有系統 401；如果是真災難就是全面停擺。

`標準` keyring 匯出 recovery key 離線保管；還原時先匯入金鑰再啟動。

`客製` 金鑰與憑證離線保管清單（含後端系統的加密金鑰），演練步驟 1 一起核對。

### 案例 22 · Enterprise：Vault 維護時，OIDC 登入突然全部失敗

**情境**：Vault 停機半小時，openid-connect 的 client_secret 用 `{vault://hcv/…}` 參照，快取 TTL 到期後 DP 取不到值。Vault 團隊不知道 Kong 依賴它。

**影響**：所有使用者登入失敗，而且發生在 Vault「維護完成」之後，很難把兩件事關聯起來。

`標準` Vault 做 HA；`resurrect_ttl` 長於預期維護窗口；必要時暫改 `{vault://env/…}`。

`客製` 依賴關係登記（Kong 依賴 Vault、Redis、IdP、PostgreSQL）並納入各自的變更通知名單；Vault 維護納入凍結。

## 13.3 管理與流程

這一組的共同點：設定都對，出事的是「沒人負責」或「沒有流程」。Kong 能提供的是留痕與工具，制度要自己建。

### 案例 23 · 管理：沒人知道某條 API 還有沒有人在用

**情境**：閘道上兩年累積 300 多條 Route，一半沒有擁有者。要下線舊版本時沒人敢動；資安問「這條 API 誰負責」答不出來。

**影響**：技術債累積、變更風險無法評估；每次改版都要「先問問看有沒有人在用」。

`標準` 每個 Service／Route／Consumer 加 `tags`（owner、system、tier）；http-log 統計 30 天無呼叫的 API；Enterprise 用工作區歸屬。

`客製` API 目錄與擁有者登記表跟 deck 檔放同一個 repo；季度盤點：無擁有者的 API 先掛 request-termination 回 410 兩週，沒人反應再刪。

### 案例 24 · 管理：金鑰申請靠 email，誰拿了哪把沒人記得

**情境**：新系統要接 API，工程師寫信給閘道管理員要一把 key，管理員在 Manager 手動建 consumer，沒登記。半年後要輪替，沒人知道這把 key 給了誰、還有沒有在用。

**影響**：金鑰無法輪替、無法撤銷；離職人員可能還握有可用金鑰；稽核不過。

`標準` Dev Portal 應用註冊（Enterprise）自助申請、自動留痕；OSS 至少 consumer 加 tags 記申請人與日期。

`客製` 申請單（系統、負責人、用途、額度、到期日）；consumer 在 deck 檔裡，Git 的 MR 就是登記簿；年度金鑰輪替流程。

### 案例 25 · 管理：多團隊互相覆蓋設定，出事沒人承認

**情境**：三個系統團隊都能改閘道，沒有 MR 審核，也沒開 audit_log。某天限流值被改小，財務批次失敗，查不到誰改的。

**影響**：跨團隊互相指責；沒有變更紀錄就沒有責任歸屬，問題會再發生。

`標準` `audit_log`（Enterprise）記錄誰改了什麼；RBAC 分工作區；OSS 至少關掉 Admin API 手改、只允許 decK。

`客製` 所有變更走 Git MR，CODEOWNERS 指定審核人；CI 是唯一能 sync 的身分；Manager 給人只讀。

### 案例 26 · 管理：稽核要「三個月前誰打了什麼」，log 只留 7 天

**情境**：客戶稽核要求提供某筆交易的 API 呼叫紀錄。http-log 有送到日誌平台，但保存期設 7 天；而且那筆是 401，log 裡沒有 consumer 欄位。

**影響**：無法回應稽核；合約可能有罰則；資安評鑑扣分。

`標準` http-log 欄位含 consumer／route／service／latencies／upstream_status；認證失敗也留 client_ip 與嘗試的身分（custom_fields_by_lua 可加）。

`客製` 日誌保存策略文件（哪些欄位、留多久、誰能查，依法規 6–12 個月）；稽核查詢的儲存查詢範本。

### 案例 27 · 管理：後端團隊把 429 當成閘道故障

**情境**：批次被限流回 429，後端團隊在群組說「閘道壞了」並開事故單。其實是他們自己把排程改到跟另一個系統同一時間。

**影響**：誤報事故、跨團隊摩擦；真正的問題（排程撞車）沒人處理。

`標準` 429 帶 `RateLimit-*` 與 `Retry-After` 標頭；exit-transformer（Enterprise）統一錯誤格式，內含說明與聯絡窗口。

`客製` 介接規格寫明額度、時段與退避方式；排程錯開表由整合窗口維護；事故分類先看回應是 Kong 回的還是後端回的（http-log 的 `source`）。

### 案例 28 · 管理：API 改版沒通知，下游半夜批次失敗

**情境**：會計系統把回應欄位改名後直接部署；三個下游系統當晚批次失敗，沒有人知道有改版。

**影響**：多系統中斷、資料補跑；下游對閘道與會計系統失去信任。

`標準` `/v1` 與 `/v2` 路由並存；舊版 request-termination 回 410 + 公告訊息；request-validator（Enterprise）擋不合規格的請求。

`客製` API 變更管理流程：OpenAPI diff → 通知消費者清單（consumer 的 tags 就是清單）→ 並存期至少一個結帳週期。

### 案例 29 · 管理：結帳期間有人改閘道設定

**情境**：月結當晚，某團隊的 CI 自動 sync 一個「小改」，順便帶進一條 Route 的 `strip_path` 變更；會計批次全部 404。

**影響**：月結延誤，隔天早上財務無法出報表。

`標準` 凍結期 CI 只跑 `deck gateway diff` 不 sync；Enterprise 可在凍結期把 CI 使用者的 RBAC 暫降為唯讀。

`客製` 凍結窗口制度（結帳前 24 小時到結帳完成）寫進變更管理；管線讀「凍結行事曆」自動擋 sync，例外要主管核准。

## 13.4 排錯順序與上線前檢核

{% hint style="success" %}
**經驗 · 排錯順序 — 五分鐘定位法**

1 看回應標頭 `X-Kong-Proxy-Latency` vs `X-Kong-Upstream-Latency`：誰慢。2 看狀態碼速查表：Kong 回的還是後端回的（http-log 的 `source` 欄位）。3 查 `/upstreams/{name}/health`：後端健不健康。4 看 prometheus 的連線數與 shared dict：閘道自己撐不住？5 `deck gateway diff`：設定有沒有漂移。多數問題在前三步就結束。
{% endhint %}

{% hint style="success" %}
**經驗 · 上線前檢核（技術） — 十六項**

Admin API 不對外 ✓ trusted_ips 設好 ✓ 每個 Service 有逾時與 retries ✓ Upstream 主動+被動健康檢查 ✓ 限流 policy 非 local（多 DP）✓ key_in_query 關 ✓ http-log 移除機密標頭 ✓ prometheus 四個 flag 開 ✓ 探針用 /status/ready ✓ PDB 與反親和 ✓ 三種憑證到期告警 ✓ 每環境一組 CP，deck 用 select-tag ✓ 授權到期告警 ✓ RBAC 開且 CI 用專屬使用者 ✓ keyring／Vault 金鑰離線保管 ✓ audit_log 匯出日誌平台 ✓
{% endhint %}

{% hint style="success" %}
**經驗 · 上線前檢核（管理） — 六項**

每條 API 有擁有者 tag ✓ 每把金鑰有申請單與到期日 ✓ 所有變更走 MR，CI 是唯一 sync 身分 ✓ 日誌保存期與稽核查詢範本定好 ✓ 凍結行事曆與例外核准流程 ✓ 依賴登記（Vault、Redis、IdP、PostgreSQL）與變更通知名單 ✓
{% endhint %}
