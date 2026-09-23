# 7 安全

- **Admin API 絕不對外**：只在 CP、只在內網、開 RBAC `Enterprise`；OSS 至少用 NetworkPolicy 與 Router 不開 Route。史上多數 Kong 事故都是 8001 暴露。
- **使用者認證**：openid-connect `Enterprise` 對接 MAS 同一個 IdP（OIDC）；OSS 用 jwt 插件驗 IdP 簽發的 token（要匯入 IdP 公鑰）。
- **系統對系統**：key-auth（金鑰輪替：同一 consumer 可掛多把，先加新再刪舊）或 mtls-auth。key-auth 的 `key_in_query` 預設開，會讓金鑰出現在 URL 與 log，正式環境關掉；`hide_credentials: true` 不把金鑰轉給後端。
- **授權**：acl 插件依 consumer 群組允許／拒絕；預設拒絕。
- **傳輸**：入口 TLS 1.2+；閘道到後端 mTLS 用 Certificate + Service 的 `client_certificate`。
- **請求大小與方法**：request-size-limiting；Route 限定 methods。
- **標頭**：Kong 會加 `X-Consumer-Id / X-Consumer-Username / X-Credential-Identifier` 給後端，後端可用但不要信任來自外部的同名標頭（Kong 會覆寫）。
