# 给 AI 的部署说明

> 复制整篇发给 AI，即可让它部署本项目的服务端。

## 任务

把这个仓库的服务端部署到公网，让两台手机都能访问。完成后把完整地址（含 `/ws`）发给我。

仓库：`https://github.com/xiaoxy498/123456/tree/main/sever.zip`

## 约束

1. 只部署，不改代码。不要修改、重写、格式化仓库里的任何已有文件，例如：
   `server/src/server.js`、`server/package.json`、`server/package-lock.json`、
   `server/test/*`、`Dockerfile`、`docker-compose.yml`、`docs/*`、`android/**`。
2. 需要 systemd、Nginx、Caddy、防火墙配置时，在仓库目录之外新建文件。
3. 不要修改协议字段、消息类型、环境变量名或默认值。
4. 不要开多副本、自动扩容、cluster；状态在进程内存，多实例会互相看不见。
5. 不要在仓库文件、提交信息、日志或回复里写任何密钥、Token、证书私钥。
6. 不要打印或保存画面帧、控制指令内容；服务端本身不落盘。
7. 端口被占用就换端口（如 `PORT=8081`），不要停系统进程。
8. 若只能选 Serverless，直接说不兼容并说明改造点，不要部署后说已完成。

## 项目事实

| 项 | 值 |
| --- | --- |
| 服务端目录 | `server/` |
| Node.js | ≥ 18 |
| 依赖 | 仅 `ws@^8`，`npm ci --omit=dev` |
| 启动 | `cd server && node src/server.js` 或 `npm start` |
| 端口 | `process.env.PORT`，默认 `8080`（WebSocket 中继） |
| 后台端口 | `process.env.ADMIN_PORT`，默认 `8090`（网页后台，与中继分开） |
| 绑定 | `0.0.0.0`（`HOST`，默认已是） |
| WebSocket 路径 | `/ws` |
| 健康检查 | `GET /health` |
| 可选环境变量 | `ROOM_TTL_MS` 默认 600000；`MAX_FRAME_BYTES` 默认 2097152 |
| 状态 | 全在内存，无数据库，重启清空 |
| 实例 | 必须单实例常驻 |

## 选方案

先探测环境，按优先级选第一个可行的：

A 平台发布能力（给出公网地址） ＞ B Docker / Compose ＞ D GitHub 导入的常驻 Web Service ＞ C 裸 Node + 反向代理 ＞ E 内网穿透 ＞ F Serverless

不要假设我用哪个平台或 AI 工具。多个可行时选优先级最高的，并说明理由。

## 步骤

1. `git clone` 仓库，只动 `server/`（不改内容，只运行）。
2. 按上面参数部署。
3. 公网使用必须 `wss://`。反向代理指向 `127.0.0.1:8080`，只开放 443，让 8080 只监听本机。
   网页后台单独走 `8090`，需要时可另配一个域名/路径反代到 `127.0.0.1:8090`。
   Nginx 必须带：

   ```nginx
   proxy_http_version 1.1;
   proxy_set_header Upgrade $http_upgrade;
   proxy_set_header Connection "upgrade";
   proxy_read_timeout 3600s;
   proxy_send_timeout 3600s;
   client_max_body_size 4m;
   ```

   漏掉 `Upgrade` / `Connection` 会表现为「连上即断开」。

4. 验收（全部必做）：
   - `GET /health` → 200，Body 含 `"ok":true`
   - `GET /` → 200
   - `/ws` 升级请求 → 101
   - `ws` 客户端发 `host.register` → 收到 `host.registered` 和 6 位配对码
   - 从服务器外部访问，不只本机 curl

5. 输出结果。

## 关于免费公共实例

应用默认已内置作者提供的公共服务器 `ws://106.14.219.48:8080/ws`，无需部署即可试用。公共实例经过第三方服务器，仅适合试用；涉及隐私请按上面的步骤自建。

## 输出格式

```text
方案：
地址：wss://<域名>/ws
健康检查：通过 / 不通过
持久化：无，重启清空
休眠：会 / 不会；冷启动约 X 秒
TLS：已启用 / 未启用
实例：单实例
日志：docker compose logs -f relay ／ journalctl -u palmcontrol -f
```

附带一句：两台手机装同一个 APK，设置里填上面的地址；被控端点「开始共享屏幕」拿到 6 位配对码，控制端输入后由被控端确认。
