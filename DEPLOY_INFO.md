# serverless-dns 部署与运维指南

本项目基于 Cloudflare Workers 部署由 [RethinkDNS](https://github.com/serverless-dns/serverless-dns) 驱动的高性能无服务器 DNS-over-HTTPS (DoH) / DNS 解析器，支持基于规则库的广告与恶意域名拦截，已成功 Fork 并部署至 Cloudflare 边缘计算平台，绑定自定义域名并配置 GitHub Actions 每日自动化同步上游、构建测试与健康验证。

---

## 核心信息速查

- **自定义主域名**：https://dd.ihot.cc.cd
- **DoH 查询端点**：
  - 标准 RFC 8484 查询：`https://dd.ihot.cc.cd/dns-query`
  - 拦截规则配置面板：`https://dd.ihot.cc.cd/` 或 `https://dd.ihot.cc.cd/configure`（跳转至 RethinkDNS 官方交互式规则生成器，生成的参数自动应用在自定义域名上）
- **GitHub 维护仓库 (Fork)**：https://github.com/GD2021/serverless-dns
- **上游官方仓库**：https://github.com/serverless-dns/serverless-dns
- **跟踪上游分支**：`main`
- **Cloudflare 账户**：`Waka` (`fc6f79e77dca61be461e2dedb82244f5`)
- **Cloudflare Zone**：`ihot.cc.cd` (`55ed57f400f283fa0d2599d50d390508`)
- **部署架构**：
  - **Worker 计算**：`serverless-dns` 运行在 Cloudflare Workers 边缘计算平台，通过 Webpack 打包 ES Module 模块化运行，启动耗时仅约 6ms；
  - **规则库与解析**：内置高效 LFU 缓存与前缀树 (Trie) 规则匹配引擎，结合上游全球分布式 DoH 节点转发与本地边缘缓存；
  - **自定义域名路由**：`dd.ihot.cc.cd` 通过 Cloudflare Workers Custom Domain 自动绑定，由 Cloudflare 权威边缘节点提供 Anycast 网络加速和全自动 Google Trust Services TLS/SSL 证书托管。

---

## 规范命名资产清单

| 资源类型 | Cloudflare 资源名称 / 域名 | 归属 / 标识 ID | 用途说明 |
|---|---|---|---|
| **Worker 服务** | `serverless-dns` | Account: `fc6f79e77dca61be461e2dedb82244f5` | 核心 DoH 解析器与分流阻断引擎 |
| **自定义域名** | `dd.ihot.cc.cd` | Zone: `ihot.cc.cd` (`55ed57f400f283fa0d2599d50d390508`) | 挂载于 Worker 的 Custom Domain，边缘代理并托管 SSL |
| **DNS 记录** | `dd.ihot.cc.cd` (AAAA `100::`) | Record ID: `3ddea6a0d39180579c1489cd12c29b45` | Cloudflare 自动托管的 Worker 路由解析记录 |
| **环境变量 (Vars)** | `LOG_LEVEL` / `WORKER_ENV` / `CLOUD_PLATFORM` | Worker 环境变量 | `LOG_LEVEL = "info"`, `WORKER_ENV = "production"`, `CLOUD_PLATFORM = "cloudflare"` |

---

## 必须配置的 Secrets 说明

| Secret 名称 | 存放位置 | 类型 | 状态与用途说明 |
|---|---|---|---|
| `CLOUDFLARE_API_TOKEN` (或 `CF_API_TOKEN`) | GitHub Actions Secrets | Secret | **需手动在 GitHub 仓库添加**：供 GitHub Actions 工作流在云端自动调用 `wrangler deploy` 自动部署。当前已通过 Cloudflare API MCP 完成首次生产部署；后续若需要 GitHub Actions 每天全自动更新发布生产，建议在 GitHub 仓库添加该 Token。 |
| `CLOUDFLARE_ACCOUNT_ID` (或 `CF_ACCOUNT_ID`) | GitHub Actions Secrets | Secret | **可选（工作流已内置兜底）**：账户 ID `fc6f79e77dca61be461e2dedb82244f5`，工作流中已内置该默认值。 |

---

## 接口访问与客户端配置示例

### 1. 浏览器 / 系统 DoH 配置

在支持 DoH (DNS over HTTPS) 的设备或软件中填入 DoH 模板 URL：

```text
https://dd.ihot.cc.cd/dns-query
```

若使用了个性化 Blocklist（通过访问 `https://dd.ihot.cc.cd/` 生成配置），URL 形式为：

```text
https://dd.ihot.cc.cd/<blockstamp>/dns-query
```

### 2. cURL RFC 8484 测试

```bash
# POST 查询 (RFC 8484 标准 wire-format)
curl -v -H "Content-Type: application/dns-message" \
     -H "Accept: application/dns-message" \
     --data-binary @- "https://dd.ihot.cc.cd/dns-query"

# GET 查询 (Base64URL 编码)
# EjQBAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE 为 example.com A 记录标准查询
curl -v -H "Accept: application/dns-message" \
     "https://dd.ihot.cc.cd/dns-query?dns=EjQBAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE"
```

---

## 自动化更新与定时机制

1. **上游自动同步**：
   - GitHub Actions 工作流：`Sync Upstream & Deploy` (`.github/workflows/sync-and-deploy.yml`)
   - 调度周期：北京时间每天 04:07（UTC 20:07，cron: `7 20 * * *`）
   - 机制：
     - 对比官方上游 `main` 分支最新 commit 与当前分支；
     - 有更新时自动识别变更内容、分析依赖与核心脚本改动、备份并保护自建部署配置；
     - 自动合并上游分支，若检测到代码冲突则自动停止并保护生产服务；
     - 运行 `./src/build/pre.sh` 更新拦截规则元数据并重新构建 `dist/worker.js`；
     - 执行端到端线上探测验证；
     - 将同步后的代码推送至 Fork 仓库主分支。
2. **手动重新部署入口**：
   - 访问 GitHub 仓库 -> `Actions` -> 选择 `Sync Upstream & Deploy` -> 点击 `Run workflow` -> 可勾选 `force_deploy` 强制重新构建与部署。
