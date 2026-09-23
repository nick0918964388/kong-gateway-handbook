# 11 維運：升級、備份、憑證

- **升級**：只用 LTS；順序 CP → DP；CP 升級要跑 `kong migrations up` 再 `finish`；DP 版本不能高於 CP；先在 SIT 演練，逐副本滾動，保留上一版映像可回退。
- **備份**：hybrid 模式真正的狀態只有 CP 的 PostgreSQL（實體設定）+ Git（宣告式來源）+ 憑證 Secret；DP 沒有狀態。PostgreSQL 每日備份，Git 本身有版本；災難時「重裝 CP + deck sync」比 restore DB 更快。
- **憑證**：三種各自到期——對外 TLS（Router 或 Kong）、CP↔DP 叢集憑證（有 `kong_data_plane_cluster_cert_expiry_timestamp` 指標）、Enterprise 授權（`kong_enterprise_license_expiry_timestamp`）。全部進告警，到期前 30 天。
- **重載**：DB-less 用 `POST /config`（或 deck sync 到 CP）不需重啟；改環境變數才需重啟 pod。
- **容量**：一個 DP pod（2 core）在簡單插件組合下可達每秒數千請求；瓶頸通常在後端。以 p95 與 CPU 60% 為 HPA 門檻。
