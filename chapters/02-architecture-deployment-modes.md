# 2 架構與部署模式

| 模式 | 設定存哪裡 | 適合 | 注意 |
|---|---|---|---|
| DB-less（宣告式） | 一個 YAML 檔（kong.yml），啟動時載入或以 /config 端點推送 | 單一環境、GitOps、PoC；資料面（DP）本質上就是 DB-less | 需要資料庫的插件不能用（例如 oauth2、依賴 DB 的 rate-limiting 政策）；改設定 = 重載整份 |
| Traditional（DB 模式） | PostgreSQL，透過 Admin API 改 | 單站小規模 | Admin API 與流量口在同一個程序；DB 掛了 Admin API 不能用（流量以快取續跑） |
| Hybrid（CP／DP 分離） | 控制面（CP）接 PostgreSQL；資料面（DP）從 CP 經 mTLS 取設定並快取 | 正式環境的標準 | 一個 CP 下所有 DP 拿同一份設定 → 每個環境一組 CP；CP 版本 ≥ DP，升級先 CP 後 DP |

**Hybrid 模式的關鍵行為**：DP 啟動時連 CP（:8005，wRPC），收到設定後寫本地快取；之後 CP 有變更就增量推送；CP 或資料庫停機時 DP 用快取繼續服務，只是不能改設定。Admin API（:8001）只在 CP 上；DP 只開流量埠（:8000／:8443）與 status 埠（:8100）。

> 客戶端 → SLB／OCP Router → DP ×N（:8000/:8443） → Upstream／Target（後端） CP ×2（:8001 Admin、:8005 叢集） ⇄ PostgreSQL
