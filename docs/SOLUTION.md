# Echo-cloud-mail — 技术方案与运维手册

> 基于 Cloudflare Workers 的一站式邮箱服务平台（上游 maillab/cloud-mail 的 fork）
> 本 fork 的部署实例：两个邮箱域名 `229830.xyz` / `kechox.com`，双 web 入口 `mail.229830.xyz` / `mail.kechox.com`

## 版本记录
| 版本 | 日期 | 变更说明 | 作者 |
|------|------|----------|------|
| v0.0.0-dev | 2026-11-25 | 建立运维手册：GitHub Actions 自动部署、双域名、发信通道配置 | Yuehuaer |

## 架构原理

```
GitHub push (main)
    │  mail-worker/** 或 mail-vue/** 变更触发
    ▼
GitHub Actions (.github/workflows/deploy-cloudflare.yml)
    │  注入 Secrets → 生成 wrangler-action.toml → pnpm wrangler deploy
    ▼
Cloudflare Worker: echo-cloud-mail
    ├── KV (namespace b3a7cc...) → setting: 缓存（运行时读）
    ├── D1 (a310e44c...)         → 用户/邮箱/设置持久化
    ├── Assets                   → mail-vue 构建产物
    └── Resend API               → 站外发信
         ▲
CF Email Routing: *@229830.xyz / *@kechox.com → Worker（收信）
```

### 各模块职责
- **mail-worker**：Cloudflare Worker 后端（Hono），处理 API / email / cron / WebSocket。
- **mail-vue**：Vue 前端，构建后作为 Worker Assets 提供。
- **GitHub Actions**：构建与部署， Secrets 提供所有域名/账号/密钥参数。
- **CF Email Routing**：把两个域名的入站邮件交给 Worker。
- **Resend**：站外发信，按发件域名查对应 API key。

### 关键数据流
- **收信**：发件方 → CF MX(route1/2/3.mx.cloudflare.net) → Email Routing 规则 → Worker email handler → D1。
- **发信**：Worker → `setting.resendTokens[发件域名]` → Resend API → 收件方。
- **配置**：D1 `setting` 表是源，`/api/init` 时 refresh 写入 KV `setting:`；Worker 运行时读 KV。

## 重建指南

### 所需环境
- Node.js 24、pnpm 11（GitHub Actions 中固定）
- Cloudflare 账户 + API Token（Workers/D1/KV/Email Routing 权限）
- Resend 账户 + 已验证域名

### 重建步骤
1. 克隆仓库
   ```bash
   git clone https://github.com/Yuehuaer/Echo-cloud-mail.git
   cd Echo-cloud-mail
   ```
2. 在 GitHub 仓库 Settings → Secrets and variables → Actions 配置（详见「关键配置」）。
3. 在 Cloudflare 侧确认资源：Worker `echo-cloud-mail`、D1 `echo-cloud-mail`、KV `echo-cloud-mail`、两个 zone 的 MX/SPF/DKIM、Email Routing 规则、自定义域名绑定。
4. 手动触发部署：`gh workflow run deploy-cloudflare.yml --ref main`，用 `gh run watch` 观察。
5. 验证：`curl https://mail.kechox.com` 应返回 200；登录页域名下拉应有 `@kechox.com` 和 `@229830.xyz`。

### 验证清单
- [ ] workflow_dispatch 全流程绿
- [ ] `mail.229830.xyz` / `mail.kechox.com` 均 HTTP 200
- [ ] 两个域名都能收到邮件
- [ ] 从 `@kechox.com` 发信到站外成功（Resend 返回 message id）
- [ ] 登录页域名顺序：kechox.com 在前

## 故障排除

### Error 1016（Origin DNS error）
- **原因**：custom domain 绑定被 `wrangler deploy` 冲掉（部署配置只声明了一个 route，wrangler 同步删除未声明绑定）。
- **排查**：查 `GET /accounts/{acc}/workers/domains` 是否有该 hostname；查 zone DNS 是否有 `AAAA 100::` proxied 记录。
- **修复**：把第二个域名写进 workflow 的 `CUSTOM_DOMAIN2` Secret，部署时自动追加 route（已实现，commit b65f928）。

### 能收信但发不出站外邮件
- **原因**：`resend_tokens` 没有该发件域名的 key；`noSendProvider` 检查在超管豁免之外。
- **排查**：Resend 后台域名是否 verified；D1 `setting.resend_tokens` 与 KV `resendTokens` 是否都含该域名。
- **修复**：Resend 验证域名 → D1 + KV 双写 key（参见会话存档 2026-11.md）。

### Email Routing 规则接口报 2020 Invalid rule operation
- **原因**：`all` matcher 的规则被锁定，连 DELETE 都失败。
- **修复**：用 `literal` matcher + 通配符值 `*@domain.com` 创建 worker 动作规则。

### 配置改了不生效
- **原因**：只改了 D1 或只改了 KV；`/api/init` 会用 D1 refresh 覆盖 KV。
- **修复**：D1 `setting.resend_tokens` 与 KV `setting:` 里的 `resendTokens` 两处都改。

### localhost 访问站点报 502 / DNS 解析失败
- **原因**：本地网络出口（代理）不稳定，非站点故障。
- **排查**：先用 CF API 确认线上绑定与 DNS 正常，再换网络环境重试。

## 关键配置

### GitHub Secrets（仓库 Settings → Secrets and variables → Actions）
| Secret | 说明 |
|--------|------|
| `NAME` | Worker 名 = echo-cloud-mail |
| `CUSTOM_DOMAIN` | 主 web 入口 = mail.229830.xyz |
| `CUSTOM_DOMAIN2` | 第二 web 入口 = mail.kechox.com（新增，防止 deploy 冲掉） |
| `DOMAIN` | 邮箱域名 JSON 数组，顺序决定登录页下拉顺序 = `["kechox.com","229830.xyz"]` |
| `ADMIN` | 超级管理员邮箱（硬编码单账号判定）= Yue@kechox.com |
| `JWT_SECRET` | 登录令牌密钥 |
| `CLOUDFLARE_API_TOKEN` | CF API Token（临时 7 天，需定期更换） |
| `CLOUDFLARE_ACCOUNT_ID` | 8dd514fc5bd82b5f4b418881abd33a46 |
| `D1_DATABASE_ID` | a310e44c-76c7-4200-88ac-59fdf9c6b722 |
| `KV_NAMESPACE_ID` | b3a7cc3066d24ebeb02d2c7aede41f65 |

### Cloudflare 资源
- Worker `echo-cloud-mail`；D1 `echo-cloud-mail`；KV `echo-cloud-mail`
- Zone `229830.xyz`（a5dcffec…）、`kechox.com`（ee38ac0a…）
- Custom domains: `mail.229830.xyz`、`mail.kechox.com` → echo-cloud-mail
- Email Routing: 两域名 MX → route1/2/3.mx.cloudflare.net；规则 `*@domain` → Worker

### 敏感信息处理
- 本仓库为公开仓库，API Token / JWT secret / Resend key 仅存 GitHub Secrets 与 CF 配置，不写入仓库文件。
- `.workbuddy/`（本地记忆）已在 `.gitignore` 排除，不推送。

## 参考链接
- 仓库: https://github.com/Yuehuaer/Echo-cloud-mail
- 上游: https://github.com/maillab/cloud-mail
- CF API: https://developers.cloudflare.com/api/
- Resend API: https://resend.com/docs/api-reference
- 会话存档: `docs/session-log/2026-11.md`
