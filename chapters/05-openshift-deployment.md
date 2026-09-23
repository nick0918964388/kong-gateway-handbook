# 5 在 OpenShift 上部署

- **安裝方式**：Helm chart（`kong/kong`）或 Kong Gateway Operator（KGO）。OpenShift 要注意 SCC：Kong 映像以非 root 執行，通常不需 anyuid；Enterprise 需要 license Secret。
- **Hybrid 於 OCP**：CP 與 DP 分別是 Deployment；CP 的 Service 開 8001／8005／8002（Manager），DP 的 Service 開 8000／8443／8100；叢集憑證放 Secret，兩邊掛同一份。
- **入口**：外部 → SLB → OCP Router → Route → DP Service（TLS 可在 Router 終結或 passthrough 給 DP）；需要 mTLS 或非 HTTP 時用 LoadBalancer Service 直接進 DP。
- **HA**：DP ≥3、podAntiAffinity、PDB minAvailable 2、HPA；readiness 用 `/status/ready`（3.x DP 在收到設定前會回 503，這是正確行為）。
- **資源**：DP 記憶體主要是 `mem_cache_size`（預設 128 MB，實體多要調大）+ 每個 worker 的 Lua VM；worker 數 = CPU 數（`nginx_worker_processes: auto`）。給 limits 時記得 nginx 看到的是節點 CPU 數，容器只給 1 core 卻開 16 個 worker 會互搶。
- **DNS**：Kong 自己解析並快取後端名稱；指向 OCP Service 名稱最穩；指向 headless Service（逐 pod IP）要縮短 `dns_stale_ttl`，否則 pod 換 IP 後短時間打到舊 IP。

```yaml
# DP 的關鍵環境變數（hybrid）
KONG_ROLE=data_plane
KONG_DATABASE=off
KONG_CLUSTER_CONTROL_PLANE=kong-cp.kong.svc:8005
KONG_CLUSTER_CERT=/etc/secrets/cluster/tls.crt
KONG_CLUSTER_CERT_KEY=/etc/secrets/cluster/tls.key
KONG_PROXY_LISTEN=0.0.0.0:8000, 0.0.0.0:8443 ssl
KONG_STATUS_LISTEN=0.0.0.0:8100
KONG_TRUSTED_IPS=10.128.0.0/14          # Router / SLB 的來源網段，讓 X-Forwarded-For 生效
KONG_REAL_IP_HEADER=X-Forwarded-For
KONG_MEM_CACHE_SIZE=512m
```
