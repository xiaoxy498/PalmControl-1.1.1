# 参与贡献
Trae AI帮助我完成。这个项目目标是"手机远控手机"的可用开源方案，优先保证**可部署、可维护、行为可预期**。

---

## 开发环境

### Android

| 依赖 | 版本 |
| --- | --- |
| JDK | 17 |
| Android SDK | Platform 35、Build-Tools 35.0.0 |
| Gradle | 使用 `android/gradlew` |
| Android Studio | 最新稳定版即可 |

```bash
cd android
./gradlew :app:assembleDebug
./gradlew :app:testDebugUnitTest
```

真机验证至少覆盖：

- Android 8 / 10 / 12 / 14 各一台（或模拟器 + 一台真机）；
- 竖屏与横屏；
- 局域网 `ws://`；
- 一台有 PIN、一台无 PIN 的被控端。

### 服务端

| 依赖 | 版本 |
| --- | --- |
| Node.js | 18+（CI 使用 20/22） |

```bash
cd server
npm install
npm test
npm start
```

测试使用 `node --test`，无需额外测试框架。

---

## 分支与提交

- 从 `main` 新建分支：`feat/xxx`、`fix/xxx`、`docs/xxx`。
- 提交信息建议使用 Conventional Commits：
  - `feat: add qr pairing`
  - `fix: handle rowStride on odd-width screens`
  - `docs: clarify ws security`
- 一个 PR 只做一件事，避免格式化噪音与功能改动混在一起。

---

## 代码规范

### Kotlin

- 4 空格缩进，遵循 Kotlin 官方风格。
- 新增协议字段必须同时更新 `Messages.kt` 与 [docs/PROTOCOL.md](docs/PROTOCOL.md)。
- 主线程不做解码/编码/IO；帧处理保持在 `ScreenStreamer` / `RemoteView` 的工作线程。
- 复用 Bitmap / ByteArrayOutputStream，新增逐帧分配需要说明理由。
- 不要在 `onDraw` 中分配对象。
- 新增字符串必须同时更新 `values/strings.xml` 与 `values-en/strings.xml`。

### JavaScript

- ESM，2 空格缩进。
- 不引入重依赖；服务端保持"单文件 + `ws`"。
- 所有入站消息先校验大小再解析。
- 新增房间状态必须考虑断连、超时和重复消息。

---

## 测试要求

- 服务端新增行为：在 `server/test/server.test.js` 增加端到端用例。
- Android 纯逻辑（坐标、尺寸、地址解析、协议解析）：增加 JVM 单元测试。
- UI / 录屏 / 无障碍相关改动：在 PR 描述中写明真机型号、Android 版本与验证步骤。

---

## 安全相关 PR

涉及以下内容的 PR 需要额外说明威胁模型：

- 鉴权、PIN、房间码生成；
- 新消息类型与转发白名单；
- TLS / 加密；
- 无障碍权限使用范围；
- 录制与存储。

请不要在公开 Issue 中贴出可直接复现的漏洞利用步骤。

---

## 文档

- 用户文档：[README.md](README.md) / [README.en.md](README.en.md)
- 架构：[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- 协议：[docs/PROTOCOL.md](docs/PROTOCOL.md)

行为变更（新增消息、修改默认值、改变限制）必须同步文档。

---

## Issue 模板建议

**Bug**

```text
环境：手机型号 / Android 版本 / App 版本 / 服务器部署方式
复现步骤：
1.
2.
3.
预期：
实际：
日志（logcat / 服务端输出）：
```

**Feature**

```text
场景：
为什么现有方案不够用：
期望行为：
是否愿意提交 PR：
```

---

## 许可证

提交的代码默认以 [MIT](LICENSE) 授权。
