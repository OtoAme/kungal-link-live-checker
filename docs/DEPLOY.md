# 构建、发布与运行

## 流水线

```
push main ─► GitHub Actions(.github/workflows/build.yml)
              ├─ test : gofmt + go vet + go test -race(铁律守卫测试在此把关)
              ├─ binaries: 交叉编译并打包 6 个平台的二进制
              └─ release: 在本仓库创建预发布 → 上传全部压缩包
```

- **触发方式**:推送到 `main`,或在 Actions 页面手动运行。
- **版本**:`main-<run_number>-<run_attempt>-<commit 前 7 位>`,绑定本次构建的完整 commit SHA。每次运行或重跑都会创建独立预发布,历史版本保留在 Releases。
- **平台**:Linux、macOS、Windows,各提供 amd64 和 arm64;Linux / macOS 为 `.tar.gz`,Windows 为 `.zip`。压缩包内的程序名为 `llc` 或 `llc.exe`。
- **构建**:Go 版本读取 `go.mod`,使用 `CGO_ENABLED=0`。项目零外部依赖,故无 `go.sum`。
- **权限**:发布任务使用当前仓库的 `GITHUB_TOKEN` 和 `contents: write` 权限创建 tag、预发布及附件。

## ⚠️ 出口 IP 隔离(架构铁律,REQUIREMENTS §7)

本服务会对网盘发**出站抓取**,出口 IP 可能被限流/封禁。**强烈建议跑在与 OAuth/身份(infra)分开的服务器**上——封 IP 不得波及登录信任根。这正是它独立成仓、不并入 infra 的原因。同机部署能跑通,但共享出口 IP,有连带风险。

## 运行二进制(PM2 / systemd,独立出口机)

把 checker 放到一台**独立服务器**(出口 IP 与 OAuth/身份分离)时,无需 Docker——它是零依赖静态二进制。文件见 `deploy/`。

布局:`/opt/link-checker/{llc, .env, logs, ecosystem.config.js}`(全部 owner=运行用户)。

```bash
# 1) Download llc-linux-amd64.tar.gz from this repository's Releases first.
tar -xzf llc-linux-amd64.tar.gz
scp llc <host>:/opt/link-checker/llc

# 2) 配置 + 密钥(chmod 600,绝不入库;ecosystem 从这里读)
#    /opt/link-checker/.env:
#      LLC_ADDR=127.0.0.1:6734      # 私网/回环;切勿 0.0.0.0
#      LLC_API_KEYS=<openssl rand -hex 24>
#      LLC_RATE_RPS=5

# 3a) PM2(与其它 Node 服务统一管理)
cd /opt/link-checker && pm2 start ecosystem.config.js && pm2 save
#  更新:scp 新 llc → pm2 reload link-checker   (切勿加 --update-env:会丢掉 .env 里的 key)
# 3b) 或 systemd(原生,无需 Node):见 deploy/link-checker.service
```

调用方:**同机**消费方(论坛与 checker 同处一台)直接 `http://127.0.0.1:6734/v1/check` + `Authorization: Bearer <key>`;**跨机**(论坛在 Dokploy/kungal-neo、checker 在 kungal-old)用 **Cloudflare Tunnel + Access**——详见 [`cloudflare-tunnel.md`](./cloudflare-tunnel.md)(old 零入站端口、Access service token 锁来源 + API key 双层)。失败/超时一律当 `unknown`。

## 本地

```bash
docker compose up --build          # docker-compose.yml(dev,本地 build,API key=devkey)
# 或不走容器:
LLC_API_KEYS=$(openssl rand -hex 16) go run ./cmd/server
```
