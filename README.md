# PalmControl · 掌控

> 手机远控手机。一台 Android 手机共享屏幕，另一台 Android 手机实时观看并操作。
> 单 APK 双角色 · 自托管中继服务器 · MIT 开源

[![License: MIT](https://img.shields.io/badge/license-MIT-38BDF8.svg)](LICENSE)
[![Android](https://img.shields.io/badge/Android-8.0%2B-3DDC84.svg)](https://developer.android.com)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933.svg)](https://nodejs.org)

[English](README.en.md) · [架构说明](docs/ARCHITECTURE.md) · [通信协议](docs/PROTOCOL.md) · [AI 自动部署](docs/AI-DEPLOY.md) · [参与贡献](CONTRIBUTING.md)

---

## 这个项目解决什么问题

Windows/macOS 之间已经有成熟远控软件，但 **手机远控手机** 一直缺一个好用的方案：

- 很多方案要求两台手机登录同一个账号，或强制走厂商云；
- 很多方案只支持"看屏幕"，不支持真正的手势注入；
- 很多方案依赖公网云服务，局域网内也要绕一圈外网，延迟高、隐私不可控；
- 很多方案需要 root、需要 ADB、需要电脑中转。

PalmControl 的做法相反：

- **一个 APK，两种角色**：装在同一台手机上，既可以是"被控端"，也可以是"控制端"；
- **自托管中继服务器**：一个 Node.js 服务（约 500 行），可跑在局域网树莓派、NAS、VPS 或 Docker 里；
- **不依赖账号体系**：6 位配对码 + 可选 PIN + 被控端人工确认；
- **真手势注入**：通过无障碍服务把点击、长按、滑动、双指手势和文字输入注入被控端；
- **局域网直连可用**：同一个 Wi-Fi 下 `ws://192.168.x.x:8080/ws` 即可，不必上公网。

> 适合场景：帮父母操作手机、远程调试自己的备用机、给客户演示 App、在另一台手机上操作挂机设备。

---

## 功能一览

### 被控端（Host）

- 一键共享本机屏幕（MediaProjection 录屏，前台服务保活）
- 6 位数字配对码，10 分钟内有效
- 可选 4–6 位配对 PIN（服务端只保存 SHA-256 摘要）
- 连接请求人工确认，30 秒未处理自动拒绝
- "自动允许连接"无人值守模式（默认关闭）
- 画质三档：流畅 480p / 均衡 720p / 高清 1080p
- 帧率三档：10 / 15 / 20 FPS
- 通知栏常驻，可随时停止共享
- 横竖屏切换自动重建采集

### 控制端（Controller）

- 输入服务器地址 + 配对码即可连接
- 全屏实时画面，自动等比缩放、黑边适配
- 直接触摸操作：单击、长按、滑动、双指手势
- 工具栏：返回 / 主页 / 多任务 / 通知栏 / 键盘
- 文字输入：把文字注入被控端当前输入框
- 实时状态：FPS、码率、画面分辨率
- 息屏、断网、被控端退出都会给出明确提示

### 服务端（Relay）

- 纯 WebSocket 中继，只做消息转发，不解析画面内容
- 房间生命周期管理：10 分钟配对超时、60 秒待审批超时
- 限流与防滥用：单帧 2 MB、文本 16 KB、输入 120 条/秒、单控制端
- 心跳检测、优雅关闭、健康检查 `/health`
- 零数据库、零账号，进程重启即清空
- `Dockerfile` + `docker-compose.yml` 开箱即用

---

## 架构

```
┌────────────────────┐        WebSocket        ┌────────────────────┐
│   被控端 Host       │  ───────────────────►  │   Relay Server      │
│  MediaProjection    │   JSON 控制消息        │   Node.js + ws      │
│  VirtualDisplay     │   + 二进制 JPEG 帧     │   房间 / 配对 / 限流 │
│  AccessibilityService│ ◄───────────────────  │                    │
└────────────────────┘        WebSocket        └────────────────────┘
                                                        ▲
                                                        │ WebSocket
                                                        ▼
                                              ┌────────────────────┐
                                              │  控制端 Controller │
                                              │  解码 / 绘制 / 手势 │
                                              └────────────────────┘
```

- 画面：被控端 `MediaProjection` → `VirtualDisplay` → `ImageReader` → JPEG → WebSocket 二进制帧。
- 控制：控制端采集触摸 → 归一化坐标 → `input.gesture` JSON → 被控端 `AccessibilityService` 注入。
- 会话：6 位房间码定位房间，服务端在配对成功后直接转发双向流量，不做任何解码。
- 为什么 v1 不用 WebRTC：不需要 STUN/TURN，跨 NAT 只需一个可访问的中继；单文件服务端更易自托管。H.264/WebRTC 在 Roadmap 中。

详细设计见 [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)，消息格式见 [docs/PROTOCOL.md](docs/PROTOCOL.md)。

---

## 快速开始

### 1. 部署服务器

任选一种。服务器需要被两台手机访问到。

> 不想自己动手？把 [docs/AI-DEPLOY.md](docs/AI-DEPLOY.md) 整篇发给任意 AI，它会替你从 GitHub 拉代码并完成部署（含 TLS、验证与排错）。

**Docker（推荐）**

```bash
git clone https://github.com/<your-name>/palmcontrol.git
cd palmcontrol
docker compose up -d
```

**Node.js 直接运行**

```bash
cd server
npm install
npm start
# PalmControl relay listening on ws://0.0.0.0:8080/ws
# PalmControl console listening on http://0.0.0.0:8090/admin
```

**验证**

```bash
curl http://127.0.0.1:8080/health
# {"ok":true,"rooms":0,"uptimeSec":1}
```

**局域网地址示例**

```text
ws://192.168.1.10:8080/ws
```

**公共实例（试用）**：应用已内置 `ws://106.14.219.48:8080/ws` 作为默认服务器，装好即可直接配对，无需自建。公共实例由作者维护，可能限速或调整；涉及隐私时请自建，并在「设置 → 服务器地址」改成自己的地址。

**公网部署**：请在前面加一层 TLS 反向代理（Nginx / Caddy / Traefik），使用 `wss://`。仓库不内置 TLS 证书，避免把私钥带进代码库。

Nginx 最小示例：

```nginx
server {
    listen 443 ssl;
    server_name palm.example.com;

    ssl_certificate     /etc/letsencrypt/live/palm.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/palm.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }
}
```

**AI 客服（可选）**：服务端内置 `POST /api/chat`，对接 DeepSeek 作为 App 内 AI 客服。

服务端已内置 `dotenv`，启动时会自动读取 `server/.env`。在 `server/.env` 中配置：

```bash
DEEPSEEK_API_KEY=sk-xxxx        # 必填，否则远控类问题返回 503，不会调用模型
DEEPSEEK_MODEL=deepseek-chat    # 可选，默认 deepseek-chat（客服场景推荐）
DEEPSEEK_BASE_URL=https://api.deepseek.com  # 可选
```

> `.env` 含敏感密钥，**不会进 APK**（它在 `server/` 目录），也已加入 `.gitignore`。部署时把它放在服务端同目录即可。

为避免滥用，客服有两层限制：

- 关键词门禁：只接受远控 / PalmControl 相关问题（连接失败、黑屏、配对码、无障碍、画质等）。数学题、闲聊等会本地直接回「暂不支持回答」，**不消耗 API 配额**。
- 系统提示词限域 + `max_tokens=300`：即使放行，模型也只回答远控使用问题。
- 转人工：用户发送「转人工」会创建请求，由管理员在 App 内允许或拒绝；允许后进入人工文字对话。
- 按 IP 限流：默认每 IP 每分钟 10 次（`chatRateLimit`）。

接口请求体：`{ "messages": [{"role":"user","content":"远控连接失败了，为什么？"}] }`，响应：`{ "type":"reply", "reply":"..." }`。

### 管理员与转人工

服务端内置管理员账号（可在 `server/.env` 修改）：

```bash
ADMIN_USERNAME=ADMIN
ADMIN_PASSWORD=ADMIN123
```

流程：

1. 用户在 App 客服页发送「转人工」→ 服务端创建转人工请求。
2. 管理员用 `ADMIN / ADMIN123` 登录 App，「我的 → 客服（管理员）」进入人工客服页，自动每 3 秒刷新，标题显示「有人要转人工」。
3. 管理员点「拒绝」→ 用户侧显示「转人工已被拒绝」；点「允许」→ 用户侧显示「转人工已被允许」。
4. 允许后点「进入对话」，双方即可实时对话（每 2 秒轮询），可发送文字、文件或图片；图片可预览，文件可保存，单个附件最大 3 MB。
5. 用户点「结束对话」后，用户侧显示「对话已结束」并自动切回 AI 客服；管理员侧提示「用户已结束对话」并自动关闭对话窗口。

消息接口支持 `{ "text":"..." }`、`{ "attachment": { "name":"...", "mime":"...", "data":"<base64>" } }`，也支持文字与附件同时发送。

管理员接口（需 `Authorization: Bearer <token>`）：

- `POST /api/admin/login` — 用账号密码换 token
- `GET /api/admin/handoffs` — 转人工请求列表
- `POST /api/admin/handoffs/:id/decision` — `{ "allow": true|false }`
- `POST /api/admin/handoffs/:id/message` — 管理员回复文字或附件
- `GET /api/admin/handoffs/:id` — 会话详情

用户侧接口：

- `GET /api/chat/handoff/:id` — 查询状态与消息
- `POST /api/chat/handoff/:id/message` — 发送文字或附件
- `POST /api/chat/handoff/:id/close` — 结束人工对话并切回 AI 客服

> **结束对话与附件需要服务端 1.0.8+**。只更新 APK 不更新服务端时，结束对话会返回 404、附件会因旧版 16 KB 请求体上限返回 413，App 会提示「服务端版本过旧」。升级方式：用新包里的 `server/src/server.js` 覆盖旧文件并重启进程（`.env` 不用改）。`GET /health` 返回的 `version` 字段可确认是否生效。

### 用户后台（网页）

账号已改为**服务端统一存储**，新注册用户会实时出现在后台，可封禁/解封、查看邮箱、发 App 消息和邮件。

- 地址：`http://<服务器地址>:8090/admin`（例如 `http://106.14.219.48:8090/admin`），端口由 `server/.env` 的 `ADMIN_PORT` 控制，默认 8090；App 的服务器地址仍用 8080，两者互不冲突：

```bash
# server/.env
PORT=8080        # App 的 WebSocket 中继端口
ADMIN_PORT=8090  # 网页后台端口
```
- 测试账号：`ADMIN / admin123`，正式发布请改 `server/.env`：

```bash
CONSOLE_USERNAME=ADMIN
CONSOLE_PASSWORD=admin123
```

- 后台功能：用户列表与搜索、封禁/解封（封禁后仍可登录、查看消息与设置，但远控和客服功能会被禁用）、发送 App 消息（App「我的 → 消息」查看）、发送邮件。
- 后台只有「状态管理」和「用户管理」两个模块，数据每 10 秒自动刷新：
  - 状态管理：服务器 CPU / 内存 / 中继流量 / 运行时间、节点在线状态与延迟、当前活跃远控连接（被控端、控制端、时长、状态）。
  - 用户管理：用户列表、搜索、封禁/解封、查看详情、发布公告（广播给全部用户）。
- 节点状态通过 `FRP_NODES` 配置，每个节点需要能访问 `/health`；不配置时默认只显示本机中继：

```bash
FRP_NODES=主节点=http://127.0.0.1:8080,香港节点=http://1.2.3.4:8080
```

- 后台接口：`GET /api/console/overview`（状态数据）、`POST /api/console/announce`（`{ "title": "...", "body": "..." }` 广播公告）。
- 邮件需要 SMTP，未配置时后台会提示「邮件服务未配置」：

```bash
SMTP_HOST=smtp.example.com
SMTP_PORT=465
SMTP_SECURE=true
SMTP_USER=no-reply@example.com
SMTP_PASS=你的授权码
SMTP_FROM=PalmControl <no-reply@example.com>
```

- 数据保存在 `server/data/store.json`（用户、密码哈希、消息），重启不丢；删除该文件即清空全部账号。
- App 注册必须填写邮箱；旧版本地账号第一次登录时会自动迁移到服务端（邮箱为空，后台显示「未填写」）。
- 账号接口：`POST /api/auth/register`、`POST /api/auth/login`、`GET /api/auth/me`、`GET /api/messages`、`POST /api/messages/:id/read`。
- 后台接口（需后台 token）：`POST /api/console/login`、`GET /api/console/users`、`POST /api/console/users/:username/ban`、`POST /api/console/users/:username/message`、`POST /api/console/users/:username/email`。

> 后台页面由中继服务直接托管，和静态官网 `website/` 完全分开；部署时 `admin/` 需与 `server/` 放在同一级目录（或用 `ADMIN_DIR` 指定）。

### 2. 构建 Android APK

环境要求：

| 项目 | 版本 |
| --- | --- |
| JDK | 17 |
| Android SDK | API 35（含 Build-Tools 35.0.0） |
| Gradle | 使用仓库自带 `gradlew`（8.14.6） |
| minSdk | 26（Android 8.0） |
| targetSdk | 35（Android 15） |

```bash
cd android
./gradlew :app:assembleDebug
# 产物：app/build/outputs/apk/debug/app-debug.apk
```

安装到手机：

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

也可以直接用 Android Studio 打开 `android/` 目录运行。

> 发布版请自行配置签名：在 `android/app/build.gradle.kts` 中加入 `signingConfigs` 与 `buildTypes.release`，再用 `./gradlew :app:assembleRelease`。仓库不包含任何密钥。

### 3. 开始使用

1. 两台手机都安装同一个 APK。
2. 打开「我的」→ 注册账号（用户名 + 密码，仅保存在本机）→ 登录。
3. 进入「我的 → 设置」→ 打开 **允许本机共享屏幕**（默认关闭），服务器地址已预填，可直接使用。
4. **被控端手机**：进入「我的设备」→ 点击 **开始共享本机屏幕** → 允许通知权限 → 系统录屏授权 → 允许无障碍服务。屏幕上会显示 6 位配对码。
5. **控制端手机**：进入「远程协助」→ 输入 6 位配对码 → 点击 **开始远控** → 对方设置了 PIN 就输入，没有则直接点 **开始连接**。
6. 被控端弹出"连接请求"，点击**允许**。
7. 控制端进入全屏画面，直接触摸操作即可。

被控端开启无障碍服务的方法：系统设置 → 无障碍 → 已下载的应用 → **掌控 · 远程操作** → 开启。

---

## 安全与隐私

这是远控软件，安全边界必须说清楚：

- **默认公共中继**：不配置任何东西时，数据经过作者维护的公共服务器，适合试用。长期使用或涉及隐私请自建 Relay，数据只走你自己的机器。
- **PIN 不以明文存储**：服务端只保存 `SHA-256(pin)`；配对码 6 位数字，10 分钟过期。
- **人工确认**：默认必须由被控端点"允许"；"自动允许"默认关闭。
- **单控制端限制**：一个房间同时只允许一个控制端，避免多人围观。
- **限流**：单帧 2 MB、文本 16 KB、输入 120 条/秒，防止恶意刷屏或撑爆带宽。
- **公网必须用 `wss://`**：`ws://` 是明文传输，只应在可信局域网使用。明文流量可能被同网段设备嗅探。
- **无障碍权限很强**：开启后应用可以注入手势、读取窗口内容。请只在你信任的 APK 上开启，并在不用时关闭。
- **没有账号系统**：也就没有云端账号被盗的问题，但也没有跨设备身份认证；安全依赖配对码 + PIN + 人工确认。
- **不收集任何数据**：应用无统计 SDK、无崩溃上报、无广告。

发现安全问题请通过 Issue 私密沟通，不要直接公开可复现的攻击细节。

---

## 已知限制

Android 平台本身的限制，不是实现缺陷：

1. **息屏后画面会变黑**：`MediaProjection` 无法采集息屏内容，被控端需要保持屏幕点亮。
2. **无法注入电源键/音量键等系统按键**：无障碍服务只能执行 BACK/HOME/RECENTS/通知栏等全局动作。
3. **部分安全页面禁止注入/截屏**：银行、支付、密码管理器等页面会设置 `FLAG_SECURE`，表现为黑屏或触摸无效。
4. **每次开始共享都要重新授权录屏**：Android 14+ 要求每次 `MediaProjection` 会话重新确认，无法永久免授权。
5. **侧载 APK 可能受"受限设置"限制**：Android 13+ 需在"应用信息 → 受限设置"中允许无障碍服务。
6. **画质/延迟受网络影响**：默认 JPEG 编码，弱网下会自动丢帧；不适合高帧率游戏串流。
7. **当前无声音传输、无文件传输、无剪贴板同步**。
8. **被控端必须是 Android 8.0+**；控制端同样。

---

## Roadmap

- [ ] H.264 硬编码 + WebRTC 传输（降低带宽、提升流畅度）
- [ ] 音频回传
- [ ] 文件拖拽/传输
- [ ] 双向剪贴板同步
- [ ] 二维码配对（免手输 6 位码）
- [ ] 多控制端只读旁观模式
- [ ] 触控板模式（相对移动，便于精细操作）
- [ ] 自定义按键映射
- [ ] 端到端加密（当前依赖 TLS 传输层）
- [ ] 内置 TLS 与证书固定

欢迎在 Issue 中讨论优先级。

---

## 项目结构

```text
palmcontrol/
├── android/                     # Android 客户端（Kotlin，单 APK 双角色）
│   ├── app/src/main/java/com/palmcontrol/
│   │   ├── controller/          # 远程画面控件（解码/绘制/手势识别）
│   │   ├── host/                # 采集、编码、输入注入
│   │   ├── net/                 # 协议消息与 WebSocket 客户端
│   │   ├── service/             # 录屏前台服务、无障碍服务
│   │   ├── ui/                  # 首页、被控端、控制端、设置、客服
│   │   └── util/                # 配置、地址规范化
│   └── app/src/test/            # JVM 单元测试
├── server/                      # Node.js 中继服务器
│   ├── src/server.js            # 房间、配对、转发、限流、AI 客服
│   ├── test/server.test.js      # 端到端协议测试
│   └── Dockerfile
├── docs/
│   ├── ARCHITECTURE.md
│   └── PROTOCOL.md
├── admin/                       # 网页用户后台（由服务端在 /admin 托管）
│   └── index.html
├── website/                     # 静态官网（纯 HTML/CSS/JS，可挂 GitHub Pages）
├── docker-compose.yml
├── CONTRIBUTING.md
└── LICENSE
```

## 发布产物

发布时拆成**四个独立压缩包**，互不混在一起：

| 包名 | 内容 | 用途 |
| --- | --- | --- |
| `PalmControl-x.y.z-app.zip` | `android/` 下的 Gradle 工程 + 构建好的 APK | 给用户安装 / 二开 Android 端 |
| `PalmControl-x.y.z-server.zip` | `server/`（含 `src/`、`test/`、`package.json`、`Dockerfile`） | 部署中继 + AI 客服 + 账号 |
| `PalmControl-x.y.z-admin.zip` | `admin/`（网页用户后台） | 放进服务端同级目录，访问 `/admin` |
| `PalmControl-x.y.z-website.zip` | `website/`（纯静态 HTML/CSS/JS） | 挂官网 / GitHub Pages |

四类产物分别对应普通用户、运维、运营后台和官网，避免一次更新被迫全量替换。

---

## 官网

`website/` 是一个零依赖的静态页面（HTML + CSS + JS），可直接用 GitHub Pages 托管：

1. 仓库 Settings → Pages → Source 选 `Deploy from a branch`；
2. 分支选 `main`，目录选 `/website`；
3. 发布前修改 `website/script.js` 顶部的 `CONFIG`，把 `repo`、`download` 换成真实地址。

本地预览：`cd website && python3 -m http.server 8000`，然后打开 `http://localhost:8000`。

---

## 测试

```bash
# 服务端：配对、PIN、拒绝、断连、健康检查
cd server && npm test

# Android：地址规范化、分辨率计算
cd android && ./gradlew :app:testDebugUnitTest
```

---

## FAQ

**Q：能不能不部署服务器，两台手机直连？**
A：v1 不支持。手机之间做 P2P 需要 NAT 穿透/信令，复杂度高。中继服务器可以跑在同一个 Wi-Fi 的路由器/NAS 上，流量不出局域网。

**Q：为什么不用 ADB？**
A：ADB 需要电脑和 USB 调试，不适合"手机控手机"的日常场景。无障碍服务是免 root 的通用方案。

**Q：控制端能看到我的通知内容吗？**
A：能。控制端看到的就是被控端屏幕，包括通知栏。共享前请自行清理敏感内容。

**Q：多人同时连接？**
A：当前只允许一个控制端，第二个会被拒绝（`host_busy`）。

**Q：为什么画面有点糊？**
A：默认"均衡"档是 720p + JPEG 62。可在被控端切换到"高清"，或在服务器带宽允许时提高帧率。

**Q：支持 iOS 吗？**
A：不支持。iOS 无法侧载这类无障碍注入应用，系统也不允许第三方应用注入触摸。

---

## 许可证

[MIT](LICENSE) © 2026 PalmControl contributors

如果你基于本项目做二次开发或商用，保留版权声明即可。欢迎提 Issue / PR。
