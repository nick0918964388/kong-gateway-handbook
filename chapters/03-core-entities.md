# 3 核心實體

- **Service** — 一個後端服務的抽象：協定、host、port、path、逾時、retries。 _例：maximo-api → http://maximo-api.mas.svc:9080_
- **Route** — 怎麼對到 Service：hosts、paths、methods、headers。一個 Service 可有多個 Route。 _例：paths ["/v1/refunds"]_
- **Upstream + Target** — Service 的 host 指向 Upstream 名稱時，由 Upstream 做負載平衡與健康檢查，Target 是實際後端位址。 _例：upstream maximo-api → 10.1.1.5:9080、10.1.1.6:9080_
- **Consumer** — 呼叫方身分（人或系統），掛認證憑證（key、JWT、憑證），可綁 ACL 群組與專屬插件。 _例：consumer budget-system_
- **Plugin** — 政策。可掛在 全域／Service／Route／Consumer／Consumer Group 上。 _例：rate-limiting 綁在 Route_
- **Certificate / SNI** — 閘道對外 TLS 憑證與對應網域；CA Certificate 給 mTLS 驗客戶端。
- **Workspace `Enterprise`** — 實體的命名空間，依系統或團隊隔離，配合 RBAC。
- **Consumer Group `Enterprise`** — 把 consumer 分群，插件可綁群組（例如不同群組不同限流或遮罩）。

## Route 比對規則（3.x 新路由器）

- paths 以 `/` 開頭是**前綴比對**；以 `~` 開頭才是**正規表示式**。3.0 之前正規式不用加 `~`，升級時最常踩到。
- 多條 Route 命中時：比對條件越多、路徑越長者優先；同分時用 `regex_priority`。
- `strip_path: true` 會把 Route 的路徑前綴去掉再送後端；`preserve_host: false` 會把 Host 換成後端的。Maximo 這類會用 Host 產生連結的後端，兩者都要想清楚。
- Route 只允許 https 而請求是 http 時，Kong 回 **426 Upgrade Required**（`https_redirect_status_code` 預設 426），不是 301。
