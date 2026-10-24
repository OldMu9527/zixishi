# 网页端多人自习室

账号登录、房间大厅、在线成员、学习计时、文字聊天，以及按上海时区周一 00:00 切周的时常排行。

## 本机运行

需要 Node.js 18+。

```bash
npm install
npm run setup
npm run dev
```

浏览器打开 http://localhost:5173

演示账号：`host` / `demo123`

## 局域网多人

1. 在这台电脑运行 `npm run dev`（前端已 `--host 0.0.0.0`，后端监听 `0.0.0.0:3001`）。
2. 查本机 IP（Windows：`ipconfig` 看 IPv4）。
3. 同一 Wi-Fi 的同学访问 `http://<你的IP>:5173`。

数据保存在 `server/data/app.db`（SQLite），同一台机器上的服务被大家共用。

## 生产构建（本地跑一遍）

```bash
npm run build          # 前端 → dist-client/，后端 → dist-server/
npm start              # 单进程同时提供网页 + API + WebSocket，默认 3001
```

健康检查：http://localhost:3001/api/health

首次启动若检测不到数据表会自动执行 `prisma db push`，无需手动迁移。

---

## 部署到国内服务器（腾讯云轻量，推荐）

面向国内用户、有摄像头/屏幕共享需求的场景首选。

### 为什么选它

| | 腾讯云轻量 | Railway / Vercel |
|---|---|---|
| 国内访问速度 | 快 | 慢/不稳定 |
| WebRTC 穿透 | 可自建 TURN，稳 | 难配 |
| 摄像头/屏幕共享 | 可靠 | 经常连不上 |
| 成本 | ~¥25-50/月固定 | 按量，波动 |
| 需要备案 | 域名要备案 | 不需要 |

**关键前提**：本项目视频走 **P2P mesh**（服务器只传信令，不中转媒体流），
所以服务器带宽压力很小，3-5Mbps 足够。真正的瓶颈是 **NAT 穿透**——
项目已内置自建 coturn 的支持（见下）。

### 1. 买机器

[腾讯云轻量应用服务器](https://cloud.tencent.com/product/lighthouse)：

- 配置：**2核2G / 2核4G**（10 人以内 2G 够用）
- 系统：**Ubuntu 22.04 或 24.04**
- 带宽：**3-5Mbps**（视频不走服务器，不用买大带宽）
- 地域：选离用户近的（广州/上海/北京）

### 2. 域名备案 + 解析

- 域名需要在腾讯云完成 **ICP 备案**（国内服务器的硬性要求）
- 备案通过后，把 A 记录解析到服务器公网 IP
- 没有域名也能跑（用 IP + 443 自签证书），但浏览器会警告，且 TURN 用 IP 不方便

### 3. 一键部署

把代码传到服务器后：

```bash
git clone <你的仓库> /opt/study-room
cd /opt/study-room
sudo bash deploy/setup-vps.sh your-domain.com
```

脚本会自动完成：

1. 安装 Docker、Nginx、certbot
2. 生成 `TURN_SECRET` 和 `ADMIN_TOKEN` 写入 `.env`
3. `docker compose up -d --build` 构建启动
4. 配 Nginx 反向代理（**含 WebSocket 支持**，Socket.IO 需要）
5. certbot 申请 HTTPS 证书并自动续期
6. 部署 coturn（STUN + TURN 中继）
7. 配 systemd 开机自启

### 4. 放行安全组端口 ⚠️

这一步**必须手动做**，去腾讯云控制台 → 实例 → 安全组：

| 协议 | 端口 | 用途 |
|---|---|---|
| TCP | 80 | HTTP + 证书签发 |
| TCP | 443 | HTTPS |
| UDP | **3478** | TURN/STUN 主通道（WebRTC 靠它） |
| TCP | 3478 | TURN over TCP 兜底 |
| TCP/UDP | 5349 | TURNS（TLS） |
| UDP | **49152-65535** | **TURN 中继端口段** |

> UDP 49152-65535 漏放是最常见的坑：不放行的话，服务端能连 STUN，
> 但中继地址算不对，表现为**对称 NAT 下视频一直连不上**。

### 5. 验证

```bash
curl https://your-domain.com/api/health
# 期望：{"ok":true,"db":"up"}

# 验证 coturn 中继是否可用
sudo apt install -y turnutils
turnutils_uclient -u test -w test -p 3478 <服务器公网IP>
# 期望出现：success, total time spent
```

### WebRTC 配置说明

服务端通过 `/api/rtc/ice` 动态下发 ICE 配置给浏览器，**TURN 凭证是临时的**
（用共享密钥 HMAC 算出来，带过期时间），不下发固定密码到前端。

```
浏览器 ──GET /api/rtc/ice──> 服务端
        <──{stun:域名, turn:域名 + 临时凭证}──
浏览器 ──ICE 协商──> STUN/TURN 服务器
浏览器 ══P2P 媒体流══> 对方浏览器   ← 视频不经过你的服务器
```

`.env` 里这几个变量控制它：

```bash
TURN_SECRET=xxx            # 必须与 coturn 的 static-auth-secret 一致
TURN_EXTERNAL_IP=1.2.3.4   # 服务器公网 IP
TURN_URL=turn:turn.你的域名:3478?transport=udp
TURN_HOST=stun:turn.你的域名:3478
TURN_TTL_SEC=3600
```

**不配也能跑**，会退回腾讯云/网易的公共 STUN。但公共 STUN 在对称 NAT 下
（很多公司网、校园网、移动网络）会「能进房间但看不到别人画面」——
自建 TURN 就是为了解决这个。

### 日常维护

```bash
cd /opt/study-room
docker compose logs -f app        # 看应用日志
docker compose restart app        # 重启
docker compose up -d --build      # 拉新代码后重新部署
systemctl status coturn           # 看 coturn
systemctl restart coturn          # 重启 coturn
```

### 定时备份与恢复

部署脚本会自动装上 cron 任务：**每天 03:17 备份一次，保留 14 天**。

```bash
ls -lh /opt/study-room-backups/app-*.db.gz   # 看备份
sudo /opt/study-room-backups/backup.sh        # 手动备份一次
tail -f /opt/study-room-backups/backup.log    # 看备份日志

# 恢复（latest = 用最新那份）
sudo bash /opt/study-room/deploy/restore-backup.sh latest
```

备份用的是 SQLite 的在线备份 API（`.backup`）而不是 `cp` —— `cp` 一个正在写入的
数据库文件可能得到损坏的库。恢复前脚本会**自动把当前数据另存一份**，
万一恢复错了还能退回去。

> ⚠️ 备份在这台服务器上，服务器挂了备份也没了。重要的话再同步一份到别处
> （另一台机器 / 腾讯云 COS）。

### 视频码率已做限制

P2P mesh 架构下，每个开摄像头的人要同时上传 N-1 份流。已内置限码率：

| 类型 | 上限 | 帧率 |
|---|---|---|
| 摄像头 | 300 kbps | 24 |
| 屏幕共享 | 800 kbps | 15 |

这样 4 人同开摄像头，每人上行 900kbps，家用宽带完全扛得住。
不限制的话一个 1080p 摄像头就能吃掉 2-3Mbps，5-6 人同开就打满上行。
浏览器会根据网络状况自动降分辨率，属于正常行为。

---

## 部署到 Railway（备选）

这个项目**不能**部署到 Vercel / Netlify —— 它们是无状态的，SQLite 的文件会被清空。
需要一个能挂持久磁盘、跑常驻 Node 进程的平台。Railway 是最简单的选择。

> ⚠️ Railway 在国内访问慢，且不好配 TURN。**国内用户为主就别选它。**

### 1. 推到 GitHub

仓库需要包含（`.gitignore` 已经排除了 `node_modules`、`dist-*`、`server/data`）：

```
Dockerfile
.dockerignore
railway.toml
.env.example
prisma/schema.prisma
shared/
server/
client/
```

### 2. 在 Railway 创建服务

1. 登录 [railway.app](https://railway.app)，`New Project` → `Deploy from GitHub repo`。
2. 选中仓库，Railway 检测到 `Dockerfile` 会自动按 Docker 方式构建。
3. 配置**必须手动做**的两件事：

   **a. 给服务加一个 Volume**（否则数据每次部署都丢）
   - 服务页 → `Volumes` → `Add Volume`
   - 挂载路径填 `/app/server/data`

   **b. 生成域名**（否则外部访问不了）
   - 服务页 → `Settings` → `Networking` → `Generate Domain`
   - 会得到类似 `https://study-room-production-xxxx.up.railway.app` 的地址

4. 首次部署后灌一次演示数据（可选）：
   ```bash
   railway run npm run db:seed
   ```

### 3. 验证

```bash
curl https://<你的域名>/api/health
# 期望：{"ok":true,"db":"up"}
```

### 关键注意事项

- **副本数必须是 1**。Railway 默认是 1，但如果你手动扩容就会出问题：
  WebSocket 长连接 + SQLite 意味着房间成员状态只存在于内存里。
  两个副本时，A 房间的人连到副本 1、B 房间的人连到副本 2，
  成员列表和时长统计就会分裂。`railway.toml` 里已固定 `numReplicas: 1`。

- **WebSocket 需要粘性会话**。Railway 的代理会保持同一客户端的连接不断开；
  若将来出现频繁断连，去服务设置里确认 WebSocket 支持已开启。

- **摄像头/麦克风需要 HTTPS**。浏览器只在安全上下文里开放 `getUserMedia`，
  Railway 的域名自带 HTTPS，没问题。**不能用 http:// 访问**，否则摄像头开不了。

- **健康检查**：`/api/health` 真查一次数据库而不是只探端口，数据库异常时能更早被发现。
  `railway.toml` 里已配。**它只在部署切换时生效，不会重启已运行的服务**。
  所以给线上加单独的监控（UptimeRobot 之类），重要一些。

---

## 省钱的两种姿势

Railway 的扣费核心是 **CPU 运行时间**。两者一起开效果最好，按预算挑用即可。

### 姿势 1：自动休眠（强烈推荐，零代码）

`railway.toml` 里已默认开启 `sleepApplication: true`，部署上去就生效：

> 5–10 分钟没有任何外部请求进来，Railway 就把容器停掉，**不计 CPU 费**。
> 有人访问时自动冷启动，第一次请求可能慢 ~30 秒（这是 Railway 返回 502 的常见原因）。

✅ 对你这个项目完全无副作用：
- SQLite 文件存在挂载卷里，休眠时 Railway 会做快照，唤醒无丢失
- WebSocket 断开后用户重连即触发冷启动，无需额外处理
- 单副本环境没有亲和性问题

想保持永远在线（不省钱但响应快），把 `sleepApplication` 改成 `false`。

### 姿势 2：备份到本地（必备的最后一道防线）

如果不小心删了服务，或者改坏了 schema，Volume 也会一起没。
**定期把数据库拉一份到本地**，出问题随时能恢复。

```bash
# 1. 设置环境变量（Railway 控制台 → Account → Tokens 生成 RAILWAY_TOKEN；
#    RAILWAY_PROJECT_ID/ENVIRONMENT_ID/SERVICE_ID 从对应页面 URL 里找）
set RAILWAY_TOKEN=xxx
set RAILWAY_PROJECT_ID=p_xxx
set RAILWAY_ENVIRONMENT_ID=xxx
set RAILWAY_SERVICE_ID=s_xxx
set RAILWAY_DOMAIN=study-room-production-xxxx.up.railway.app
set ADMIN_TOKEN=xxx                # 与 Railway 服务里的 ADMIN_TOKEN 一致

# 2. 备份（首次冷启动可能要等几十秒，脚本里有 90s 超时）
node scripts/backup-railway-db.mjs ./backup-2026-10-21.db

# 3. 恢复（覆盖前会自动再备份一次当前库，强制使用 -y）
node scripts/restore-railway-db.mjs ./backup-2026-10-21.db
```

⚠️ **不要在浏览器里访问 `/api/admin/db-dump`**：那是给脚本调用的，鉴权是 Bearer 不是 cookie，浏览器没存就直接 401。

**Serverless + 按时手动备份** 基本上能把月成本压到只要 Volume 费。
如果还要更省，去 Railway 控制台里 **Volume → Trash Volume 后再删服务**，彻底停掉计费——**这之前先备份一次**。

---

## 说明

- 只有点「开始自习」后由服务端累计时长，客户端不能自己上报。
- 离开房间、点休息或断线约 15 秒会结算本段。
- 超过约 10 分钟没有心跳会自动休息（方便切去看资料页）。