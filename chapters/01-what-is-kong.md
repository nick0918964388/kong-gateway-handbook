# 1 Kong 是什麼、不是什麼

Kong Gateway 是建立在 NGINX／OpenResty（C + LuaJIT）上的 API 閘道：所有 API 請求先經過它，再轉給後端。它做的事只有一類——**在請求進出時套用政策**：認證、授權、限流、逾時與健康檢查、轉換、日誌與指標。政策以「插件」形式掛在不同範圍上。

- **是**：反向代理 + 政策執行點 + 流量可觀測點。核心（Kong Gateway OSS）為 Apache 2.0 開源。
- **不是**：不是應用伺服器、不是 ESB（不做複雜編排）、不是 WAF（可搭配）、不是身分提供者（它驗 token，不發 token；Enterprise 的 openid-connect 也是對接 IdP）。
- **版本與授權**：OSS 免費；Enterprise 加 Workspaces、RBAC、Dev Portal、進階插件與原廠支援；Konnect 是 Kong 代管的控制面 SaaS。Vitals 自 Enterprise 3.5 起不再包含。

{% hint style="info" %}
心智模型：一個請求 = **Route 對到 → 找到 Service → 經 Upstream 選一台 Target → 途中依序執行插件**。看不懂任何設定時，回到這句話。
{% endhint %}
