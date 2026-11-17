# 架构说明

本文说明 PalmControl 的整体设计、线程模型、关键流程与取舍。协议细节见 [PROTOCOL.md](PROTOCOL.md)。

---

## 1. 总览

PalmControl 由三部分组成：

| 组件 | 技术栈 | 职责 |
| --- | --- | --- |
| Android 客户端 | Kotlin / ViewBinding / OkHttp | 单 APK，同时实现被控端与控制端 |
| Relay Server | Node.js / `ws` | 房间管理、配对、双向转发、限流 |
| 文档与部署 | Markdown / Docker | 自托管说明与镜像 |

```text
┌─────────────────────┐                ┌─────────────────────┐
│  Host (Android)     │                │ Relay Server        │
│                     │  ws /ws        │                     │
│ MediaProjection     │ ─────────────► │ rooms: Map<code>    │
│ VirtualDisplay      │  text: JSON    │  host / controller  │
│ ImageReader         │  binary: JPEG  │  pending / session  │
│ ScreenStreamer      │                │                     │
│ AccessibilityService│ ◄───────────── │                     │
│ InputInjector       │                │                     │
└─────────────────────┘                └─────────────────────┘
                                                 ▲
                                                 │ ws /ws
                                                 ▼
                                       ┌─────────────────────┐
                                       │ Controller (Android)│
                                       │ RemoteView          │
                                       │ 解码 / 绘制 / 手势   │
                                       └─────────────────────┘
```

设计目标：

1. **部署简单**：一个 Node 进程即可自托管，不依赖数据库、STUN/TURN、对象存储。
2. **隐私可控**：数据只经过用户自己的服务器，Relay 不解析画面。
3. **单 APK**：降低分发成本，用户装一次即可扮演两种角色。
4. **可演进**：协议是纯 JSON + 二进制帧，后续可替换编码/传输而不动 UI。

---

## 2. Android 模块划分

```text
com.palmcontrol
├── App.kt                      应用入口
├── ui/
│   ├── MainActivity.kt         底部导航（远程协助 / 我的设备 / 我的）、本地账号、配对入口
│   ├── HostActivity.kt         被控端界面：授权、配对码、批准弹窗
│   ├── ControllerActivity.kt   控制端界面：连接、全屏画面、工具栏
│   └── SettingsActivity.kt     服务器地址、设备名、PIN、共享开关、画质/帧率
├── service/
│   ├── ScreenCaptureService.kt 前台服务：录屏 + WebSocket + 状态机
│   └── RemoteAccessibilityService.kt  无障碍服务单例
├── host/
│   ├── ScreenStreamer.kt       采集与 JPEG 编码
│   ├── StreamPresets.kt        画质/帧率预设与尺寸计算
│   └── InputInjector.kt        手势/按键/文字注入
├── controller/
│   └── RemoteView.kt           JPEG 解码、等比绘制、触摸→手势
├── net/
│   ├── Messages.kt             协议常量与 JSON 编解码
│   └── RelayClient.kt          OkHttp WebSocket 薄封装
└── util/
    ├── Prefs.kt                SharedPreferences 配置
    ├── AccountStore.kt         本地账号（PBKDF2 加盐哈希，仅存本机）
    └── ServerUrl.kt            服务器地址规范化
```

### 2.1 被控端状态机

`ScreenCaptureService.State`：

```text
Idle ──start──► Connecting ──host.registered──► Waiting(code)
                    │                              │
                    │                    join.request│
                    │                              ▼
                    │                   PendingApproval(controllerId, name)
                    │                              │ allow
                    │                              ▼
                    └──────────── fail ──────── Controlling(peerName)
                                                   │ peer.left / session.end
                                                   ▼
                                                Waiting(code)
```

- `Idle`：未共享。
- `Connecting`：已启动前台服务，等待服务器分配房间码。
- `Waiting`：持有配对码，等待控制端加入。
- `PendingApproval`：收到 `join.request`，30 秒内未处理则自动拒绝。
- `Controlling`：已配对，开始采集并转发。
- `Failed`：录屏被系统终止、服务器错误、网络断开等；服务自动停止。

被控端的 `autoAccept` 打开后，`join.request` 直接回复允许，跳过 `PendingApproval`。

### 2.2 控制端状态

控制端没有独立的状态机类，由 `ControllerActivity` 的 `paired` 标志和 UI 面板切换表达：

```text
Connect panel ──controller.join──► waiting_approval ──paired──► remote panel
      ▲                                                              │
      └──────────────── error / peer.left / session.end ─────────────┘
```

---

## 3. 画面采集与编码

### 3.1 管线

```text
MediaProjection
   │ createVirtualDisplay(targetWidth, targetHeight, densityDpi, AUTO_MIRROR)
   ▼
VirtualDisplay ──► ImageReader (RGBA_8888, maxImages = 2)
   │ acquireLatestImage()
   ▼
ByteBuffer ──► Bitmap (ARGB_8888)
   │ 处理 rowStride 填充，必要时裁掉 padding
   ▼
Bitmap.compress(JPEG, quality) ──► ByteArray ──► WebSocket binary
```

关键实现（`ScreenStreamer.kt`）：

- **由系统缩放**：`VirtualDisplay` 直接以目标分辨率创建，GPU 负责缩放，避免逐帧 CPU 缩放。
- **处理 rowStride**：部分设备的 `Image` 行跨度大于宽度，先用 `copyPixelsFromBuffer` 到 padded Bitmap，再裁切到有效区域。
- **节流**：`frameIntervalMs = 1000 / fps`，不足间隔的帧直接丢弃（`acquireLatestImage` 只取最新帧，本身也会丢旧帧）。
- **复用对象**：`Bitmap` 与 `ByteArrayOutputStream` 跨帧复用，减少 GC 抖动。
- **统计**：每秒回调一次实际 FPS 与字节数。

### 3.2 画质预设

| 档位 | 目标宽度 | JPEG 质量 | 说明 |
| --- | --- | --- | --- |
| 流畅 | 480 | 45 | 弱网/省流量 |
| 均衡（默认） | 720 | 62 | 推荐 |
| 高清 | 1080 | 78 | 高带宽 |

高度按原屏幕宽高比缩放（整数除法），并且**不会放大**小屏设备：

```kotlin
fun outputSize(srcWidth: Int, srcHeight: Int, maxWidth: Int): IntArray
```

帧率：10 / 15 / 20 FPS。

### 3.3 生命周期

- 必须在 `startForeground(..., FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION)` **之后**调用 `MediaProjectionManager.getMediaProjection()`，否则 Android 14+ 会抛出异常。
- `MediaProjection.Callback.onStop()` 触发时视为会话结束（用户从系统 UI 停止录屏）。
- 每次开始共享都需要重新走系统录屏授权；Android 不允许永久保留投影权限。
- 横竖屏变化触发 `onConfigurationChanged`，先 `stopStreaming()` 再 `startStreaming()` 重建 VirtualDisplay。
- `ScreenStreamer.stop()` 在采集线程释放 ImageReader / VirtualDisplay / Bitmap，并 `join(800ms)`，避免泄漏。

---

## 4. 输入注入

`InputInjector` 依赖 `RemoteAccessibilityService` 提供的无障碍能力。

### 4.1 坐标映射

控制端发送的是**归一化坐标**（相对画面内容区域 0..1），不是像素：

1. 控制端 `RemoteView` 计算 letterbox 后的内容矩形；
2. 触摸点映射为 `(x / contentWidth, y / contentHeight)`，clamp 到 0..1；
3. 被控端用 `displayMetrics` 换算回像素坐标。

这样控制端屏幕分辨率、比例、黑边都不影响落点。

### 4.2 支持的操作

| 操作 | 实现 |
| --- | --- |
| 单击 | `GestureDescription`，80 ms |
| 长按 | `GestureDescription`，650 ms |
| 滑动 | 多点 Path + 总时长（120–5000 ms） |
| 双指 | 两笔并行 Path，用于缩放/双指滑动 |
| IME 返回/主页/多任务/通知栏/快捷设置 | `performGlobalAction` |
| 文字 | 聚焦节点 `ACTION_SET_TEXT`，失败回退剪贴板粘贴 |

限制：

- 手势路径最多 10 笔、每笔点数在协议端有限制，防止恶意数据撑爆注入器。
- 无法注入电源键、音量键；无法绕过 `FLAG_SECURE` 页面。
- 息屏时录屏本身无画面，注入也无意义。

---

## 5. 控制端绘制与手势识别

`RemoteView` 是一个自定义 `View`：

- **解码线程**：单线程 `ExecutorService`，`AtomicBoolean` 保证同一时刻只解码一帧；解码跟不上时直接丢弃新帧（背压策略）。
- **绘制**：`onDraw` 中按 `min(width/bw, height/bh)` 等比缩放并居中，黑边填充。
- **触摸识别**：
  - `ACTION_DOWN` 记录起点；
  - 移动超过 `touchSlop` 记为滑动，每 16 ms 采样一点；
  - `ACTION_POINTER_DOWN` 进入双指模式，抬起时生成两笔手势；
  - 抬手时根据位移与时长判定单击 / 长按 / 滑动。
- **内存**：替换帧时 recycle 旧 Bitmap；`onDetachedFromWindow` 关闭解码线程并回收。

---

## 6. 服务端设计

`server/src/server.js` 是单文件实现，核心是一个 `Map<code, Room>`：

```js
Room = {
  code, host, hostName, pinHash,
  createdAt, expiryTimer,
  pending,        // { id, ws, name, timer }
  controller,     // { id, ws, name }
  sessionId
}
```

### 6.1 房间生命周期

| 阶段 | 规则 |
| --- | --- |
| 创建 | `host.register` 生成 6 位随机码（30 次尝试避免碰撞） |
| 等待配对 | 10 分钟 TTL（`ROOM_TTL_MS`），超时关闭 host |
| 待审批 | 60 秒（`pendingTtlMs`），超时通知控制端 `join_timeout` |
| 已配对 | 取消 TTL；host 断开则销毁房间 |
| 控制端断开 | 清空 session，重新开始 10 分钟 TTL |
| 主动结束 | `session.end` 通知对端并关闭控制端，恢复 TTL |

### 6.2 转发规则

- host → controller：`video.config`、`input.*` 之外的消息按白名单/直接转发；二进制 JPEG 直接透传。
- controller → host：只允许 `input.gesture`、`input.key`、`input.text`、`session.end`。
- 未配对时 host 只能发 `join.decision`，其他消息丢弃。
- 所有入站文本先校验大小（16 KB），再 `JSON.parse`，失败返回 `bad_request`。

### 6.3 限流与防护

| 限制 | 默认值 | 说明 |
| --- | --- | --- |
| 单帧大小 | 2 MB | 超出直接丢弃，不转发 |
| 文本大小 | 16 KB | 超出关闭连接（1009） |
| 控制输入 | 120 条/秒 | 超出静默丢弃 |
| 控制端数量 | 1 | 第二个返回 `host_busy` |
| 未注册连接 | 10 秒 | 未注册则关闭（4408） |
| 心跳 | 30 秒 | ping/pong，失联终止 |

### 6.4 优雅关闭

`close()` 依次：清理定时器 → 终止所有 WebSocket → 关闭 WSS → `closeAllConnections()` → 关闭 HTTP。`closeAllConnections` 用于避免已升级连接让 `httpServer.close()` 永久等待（测试中发现的问题）。

---

## 7. 安全模型

| 层 | 机制 |
| --- | --- |
| 网络 | 局域网 `ws://`；公网建议 `wss://`（反向代理提供 TLS） |
| 房间 | 6 位随机码，10 分钟有效，无持久化 |
| 身份 | 可选 4–6 位 PIN，服务端存 SHA-256，不存明文 |
| 授权 | 被控端人工确认（30 秒超时），或显式开启自动接受 |
| 并发 | 单控制端，避免围观 |
| 资源 | 帧/文本/频率限制 |
| 数据 | 无账号、无遥测、无落盘 |

明确不提供的保护：

- 无端到端加密（依赖 TLS 传输层）；
- 无设备指纹/双向证书；
- 不防御已被攻陷的服务器（Relay 可见明文元数据与画面）。

威胁模型假设：服务器由用户自己控制，局域网可信，公网部署时使用有效 TLS。

---

## 8. 关键取舍

### 8.1 中继 + JPEG，而不是 WebRTC

优点：

- 无 STUN/TURN，跨 NAT 只需一个 HTTP 反向代理；
- 服务端约 500 行，任何 VPS/树莓派都能跑；
- 客户端无 Native 依赖，APK 小、构建简单；
- 调试直观：JSON 可读，帧可直接保存为 JPEG。

缺点：

- 带宽高于 H.264；
- 无拥塞控制/自适应码率（仅靠跳帧）；
- 延迟受 JPEG 编解码与网络抖动影响。

结论：v1 优先"可部署、可维护"，H.264/WebRTC 列为 Roadmap。

### 8.2 无障碍而非 ADB/root

- 无障碍是免 root、免电脑的通用方案；
- 代价是用户必须手动开启，且受系统安全策略限制；
- ADB 方案需要 USB 调试与电脑，不适合日常"手机控手机"。

### 8.3 单 APK 双角色

- 减少分发与版本管理成本；
- 代码体量可控（UI 层两个 Activity，网络层共用 `RelayClient`/`Messages`）；
- 也方便一台手机同时装两个实例做自测。

### 8.4 服务端不保存状态

- 进程重启即清空，天然避免隐私残留；
- 代价是无法做断线重连恢复会话；
- v1 接受该取舍，会话短暂且可重新配对。

---

## 9. 性能与资源

| 项目 | 量级 |
| --- | --- |
| 720p / 15 FPS / JPEG 62 | 约 2–8 Mbps（视画面复杂度） |
| 480p / 10 FPS / JPEG 45 | 约 0.5–2 Mbps |
| 1080p / 20 FPS / JPEG 78 | 约 8–20 Mbps |
| 采集线程 | 1 个 HandlerThread |
| 解码线程 | 1 个单线程 Executor |
| 服务端内存 | 每房间仅保存引用与少量字符串 |

画面复杂度（文字/视频/游戏）对 JPEG 体积影响很大，README 中给出的数字是经验区间而非承诺。

---

## 10. 可演进方向

1. **编码**：MediaCodec H.264 → 容器化帧 → 控制端 MediaCodec 解码。
2. **传输**：WebRTC DataChannel / RTP，引入自适应码率与丢包恢复。
3. **配对**：二维码承载 `server + code + pin`，减少手输错误。
4. **安全**：端到端加密（X25519 + AES-GCM），服务器只转发密文。
5. **多端**：只读旁观、多人协作标注。
6. **输入**：相对触控板模式、组合键、自定义映射。
