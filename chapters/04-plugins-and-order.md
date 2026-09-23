# 4 插件與執行順序

每個插件宣告自己在哪個階段跑、優先序多少；請求進來時 Kong 依固定順序執行。理解順序才知道「為什麼限流在認證之後」「為什麼遮罩看得到 consumer」。

> certificate → rewrite → access（認證 → 授權 → 限流 → 轉換請求） → 代理到後端 → header_filter → body_filter（回應轉換／遮罩） → log（http-log、prometheus 計數）

| 類別 | OSS | Enterprise 加值 |
|---|---|---|
| 認證 | key-auth、basic-auth、jwt、hmac-auth、ldap-auth、mtls（CA） | openid-connect、mtls-auth、jwt-signer、vault-auth |
| 授權 | acl、ip-restriction、bot-detection | opa、route-by-header、consumer groups 綁插件 |
| 流量控制 | rate-limiting、request-size-limiting、request-termination、proxy-cache | rate-limiting-advanced（Redis 叢集級、滑動視窗）、proxy-cache-advanced、graphql-rate-limiting |
| 轉換 | request-transformer、response-transformer、correlation-id | request/response-transformer-advanced、jq、exit-transformer、request-validator |
| 觀測 | prometheus、http-log、file-log、tcp/udp/syslog-log、opentelemetry、zipkin、statsd | statsd-advanced、kafka-log |

## 同名插件的優先權（最常誤解）

同一個插件在多個範圍都有設定時，**只有一個實例會執行，不是合併**。優先序由高到低：Consumer + Route + Service → Consumer + Route → Consumer + Service → Route + Service → Consumer → Route → Service → 全域。例如全域掛 rate-limiting 100 rps、某 Route 掛 10 rps，該 Route 只套 10 rps，全域那個對它不生效。

```yaml
# 常見寫法：全域觀測、工作區認證、路由例外
plugins:
- name: prometheus            # 全域
  config: { status_code_metrics: true, latency_metrics: true }
- name: key-auth              # 綁 service
  service: maximo-api
- name: rate-limiting         # 綁 route，只對這條生效
  route: refunds
  config: { minute: 600, policy: local }
```
