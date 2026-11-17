# 通信协议

PalmControl 的客户端与服务端通过 **WebSocket** 通信，端点固定为 `/ws`。

- 文本消息：JSON（UTF-8），用于控制、配对和状态同步。
- 二进制消息：被控端发出的 JPEG 帧，服务端原样转发给控制端。
- 服务器只做房间管理与转发，不解析 JPEG，也不修改 JSON（`session.end` 等除外）。

> 版本：v1（对应 `applicationId com.palmcontrol`，versionName 1.1.2）

---

## 1. 连接与角色

```text
ws://host:8080/ws
```

连接建立后 10 秒内必须发送 `host.register` 或 `controller.join`，否则服务端返回 `error` 并关闭连接（close code `4408`）。

一个连接只能有一个角色，重复注册返回 `already_registered` / `already_joined`。

---

## 2. 消息类型总表

### 2.1 客户端 → 服务端

| 类型 | 发送方 | 说明 |
| --- | --- | --- |
| `host.register` | 被控端 | 注册房间，申请配对码 |
| `controller.join` | 控制端 | 使用配对码加入房间 |
| `join.decision` | 被控端 | 批准/拒绝控制端请求 |
| `video.config` | 被控端 | 通知控制端当前画面参数（服务端透传） |
| `input.gesture` | 控制端 | 手势注入 |
| `input.key` | 控制端 | 全局按键 |
| `input.text` | 控制端 | 文字输入 |
| `session.end` | 双方 | 主动结束会话 |
| 二进制帧 | 被控端 | JPEG 画面 |

### 2.2 服务端 → 客户端

| 类型 | 接收方 | 说明 |
| --- | --- | --- |
| `host.registered` | 被控端 | 返回 6 位配对码 |
| `join.pending` | 控制端 | 已收到加入请求，等待被控端审批 |
| `join.request` | 被控端 | 有控制端请求加入 |
| `paired` | 双方 | 配对成功，下发 `sessionId` |
| `peer.left` | 双方 | 对端断开 |
| `session.end` | 双方 | 会话结束 |
| `video.config` | 控制端 | 透传 |
| `input.*` | 被控端 | 透传 |
| `error` | 双方 | 错误信息 |
| 二进制帧 | 控制端 | 透传 JPEG |

---

## 3. 消息定义

### 3.1 `host.register`

被控端注册房间。`pin` 可为空字符串；非空时必须是 4–6 位数字。

```json
{
  "type": "host.register",
  "deviceName": "Pixel 7",
  "pin": "1234"
}
```

服务端响应：

```json
{
  "type": "host.registered",
  "code": "482913",
  "expiresInSec": 600
}
```

- `code`：6 位数字，用于配对。
- `expiresInSec`：房间在未被控制端加入前的存活时间，默认 600 秒。

错误：

| code | 含义 |
| --- | --- |
| `invalid_pin` | PIN 不是 4–6 位数字 |
| `server_busy` | 30 次尝试仍无法分配房间码 |
| `already_registered` | 当前连接已注册 |

---

### 3.2 `controller.join`

控制端加入房间。房间不存在、已过期、PIN 错误或已有控制端时立即返回 `error`。

```json
{
  "type": "controller.join",
  "code": "482913",
  "pin": "1234",
  "deviceName": "Galaxy S23"
}
```

服务端先给控制端：

```json
{ "type": "join.pending" }
```

同时给被控端：

```json
{
  "type": "join.request",
  "controllerId": "a1b2c3d4e5f6",
  "deviceName": "Galaxy S23"
}
```

若被控端开启了自动接受，会立即回 `join.decision`，控制端随之收到 `paired`。

错误：

| code | 含义 |
| --- | --- |
| `room_not_found` | 配对码不存在或已过期 |
| `invalid_pin` | PIN 缺失或不匹配 |
| `host_busy` | 已有待审批请求或已配对控制端 |
| `join_timeout` | 被控端 60 秒内未处理（服务端 `pendingTtlMs`） |
| `already_joined` | 当前连接已加入房间 |

---

### 3.3 `join.decision`

被控端对 `join.request` 的答复。`controllerId` 必须与请求一致，否则返回 `stale_decision`。

```json
{
  "type": "join.decision",
  "controllerId": "a1b2c3d4e5f6",
  "allow": true,
  "reason": ""
}
```

允许时，双方各收到一条 `paired`：

```json
{
  "type": "paired",
  "role": "host",
  "peerName": "Galaxy S23",
  "sessionId": "9f8e7d6c5b4a3210"
}
```

```json
{
  "type": "paired",
  "role": "controller",
  "peerName": "Pixel 7",
  "sessionId": "9f8e7d6c5b4a3210"
}
```

拒绝时，控制端收到：

```json
{
  "type": "error",
  "code": "denied",
  "message": "Not now"
}
```

随后服务端以 close code `4403` 关闭控制端连接。

---

### 3.4 `video.config`

被控端在开始采集后发送，服务端透传给控制端，用于显示分辨率信息。

```json
{
  "type": "video.config",
  "width": 720,
  "height": 1600,
  "fps": 15,
  "quality": 62
}
```

- `width` / `height`：实际编码尺寸（可能因屏幕比例而非 16:9）。
- `quality`：JPEG 质量（45 / 62 / 78）。

---

### 3.5 `input.gesture`

坐标全部为 **归一化 0..1**，相对被控端画面内容区域（不含黑边）。

```json
{
  "type": "input.gesture",
  "strokes": [
    {
      "start": 0,
      "duration": 80,
      "points": [[0.5, 0.5]]
    }
  ]
}
```

- `strokes`：最多 10 笔（更多会被服务端转发，客户端解析时截断）。
- `points`：每笔至少 1 个点；单点 + 短时长表示点击，单点 + 长时长表示长按。
- `start`：相对手势开始的毫秒偏移，范围 0–60000。
- `duration`：该笔持续时间，范围 1–60000 ms。
- 实际注入受 `GestureDescription.getMaxStrokeCount()` 与 `getMaxGestureDuration()` 限制。

示例：滑动

```json
{
  "type": "input.gesture",
  "strokes": [
    {
      "start": 0,
      "duration": 300,
      "points": [[0.5, 0.8], [0.5, 0.7], [0.5, 0.6], [0.5, 0.4]]
    }
  ]
}
```

示例：双指放大

```json
{
  "type": "input.gesture",
  "strokes": [
    { "start": 0, "duration": 200, "points": [[0.4, 0.5], [0.3, 0.5]] },
    { "start": 0, "duration": 200, "points": [[0.6, 0.5], [0.7, 0.5]] }
  ]
}
```

---

### 3.6 `input.key`

```json
{ "type": "input.key", "key": "back" }
```

支持的值：

| key | 动作 |
| --- | --- |
| `back` | 返回 |
| `home` | 主页 |
| `recents` | 多任务 |
| `notifications` | 展开通知栏 |
| `quick_settings` | 展开快捷设置 |

未知值会被客户端忽略。

---

### 3.7 `input.text`

```json
{ "type": "input.text", "text": "你好" }
```

- 客户端发送前会截断到 4096 字符（`Messages.text`）。
- 被控端优先对焦点节点执行 `ACTION_SET_TEXT`；失败时回退到剪贴板 + `ACTION_PASTE`。
- 若没有焦点输入框，操作静默失败。

---

### 3.8 `session.end`

任意一方主动结束：

```json
{ "type": "session.end", "reason": "user" }
```

服务端行为：

- 转发 `session.end` 给对端；
- 将控制端移出房间并关闭其连接（close code `4000`）；
- 被控端房间恢复 10 分钟配对 TTL，可再次配对。

---

### 3.9 `peer.left`

对端断开时由服务端发送。

```json
{ "type": "peer.left", "reason": "Host disconnected." }
```

常见触发：

- 被控端 WebSocket 关闭 → 控制端收到 `peer.left`，房间销毁；
- 控制端关闭 → 被控端收到 `peer.left`，房间重新进入等待配对。

---

### 3.10 `error`

```json
{ "type": "error", "code": "invalid_pin", "message": "Pairing PIN is incorrect." }
```

错误码全集：

| code | 触发场景 |
| --- | --- |
| `bad_request` | JSON 无法解析或缺少 `type` |
| `not_registered` | 未注册就发送其他消息；或 10 秒内未注册 |
| `already_registered` | 重复 `host.register` |
| `already_joined` | 重复 `controller.join` |
| `invalid_pin` | PIN 格式错误或不匹配 |
| `room_not_found` | 配对码不存在/过期 |
| `room_expired` | 房间 TTL 到期 |
| `host_busy` | 房间已有待审批请求或控制端 |
| `server_busy` | 无法分配配对码 |
| `join_timeout` | 待审批超时 |
| `denied` | 被控端拒绝 |
| `stale_decision` | `controllerId` 已失效 |
| `host_left` | 被控端在审批前断开 |
| `server_error` | 服务端内部异常 |

---

## 4. 二进制帧

- 方向：仅 **被控端 → 服务端 → 控制端**。
- 内容：完整 JPEG（以 `FF D8` 开头）。
- 大小：单帧 ≤ 2 MB（`MAX_FRAME_BYTES`），超出时服务端直接丢弃。
- 控制端不发送二进制消息；服务端忽略控制端发来的二进制。
- 未配对或没有 `sessionId` 时，服务端不转发二进制。

客户端不需要额外的帧头：JPEG 自带尺寸，控制端直接 `BitmapFactory.decodeByteArray`。

---

## 5. 完整时序

```text
Host                        Relay                         Controller
 │                            │                               │
 │── host.register ──────────►│                               │
 │◄─ host.registered(code) ───│                               │
 │                            │◄──── controller.join ─────────│
 │                            │──── join.pending ────────────►│
 │◄─ join.request ────────────│                               │
 │── join.decision(allow) ───►│                               │
 │◄─ paired ──────────────────│──── paired ──────────────────►│
 │                            │                               │
 │── video.config ────────────│──── video.config ────────────►│
 │── JPEG ───────────────────►│──── JPEG ────────────────────►│
 │                            │◄─── input.gesture ────────────│
 │◄─ input.gesture ───────────│                               │
 │                            │                               │
 │── session.end / close ────►│──── peer.left / session.end ─►│
```

---

## 6. 限制常量

| 常量 | 默认值 | 来源 |
| --- | --- | --- |
| 房间 TTL | 600 000 ms | `ROOM_TTL_MS` |
| 待审批超时 | 60 000 ms | `pendingTtlMs` |
| 被控端本地审批超时 | 30 000 ms | `ScreenCaptureService.JOIN_TIMEOUT_MS` |
| 单帧上限 | 2 MB | `MAX_FRAME_BYTES` |
| 文本消息上限 | 16 KB | `maxTextBytes` |
| 控制输入频率 | 120 条/秒 | `maxInputPerSecond` |
| 未注册宽限期 | 10 000 ms | 服务端 |
| 心跳 | 30 000 ms | 服务端 ping/pong |

环境变量：

```bash
PORT=8080
HOST=0.0.0.0
ROOM_TTL_MS=600000
MAX_FRAME_BYTES=2097152
```

---

## 7. WebSocket 关闭码

| code | 场景 |
| --- | --- |
| `1009` | 文本消息超过 16 KB |
| `4000` | 会话结束（`session.end`） |
| `4001` | 房间过期 / 被控端断开 |
| `4401` | 被控端在配对过程中离开 |
| `4403` | 被控端拒绝请求 |
| `4404` | 待审批超时 |
| `4408` | 连接后 10 秒未注册角色 |

---

## 8. 客户端兼容性约定

- 服务端必须忽略未知 `type` 的文本消息（当前实现：host 侧仅在已配对时透传，controller 侧仅白名单）。
- 客户端应容忍 `paired` 中缺失 `sessionId`（不用于鉴权，仅作会话标识）。
- `video.config` 缺失时控制端仍应正常显示画面（仅少一个分辨率标签）。
- 新增字段不得破坏旧客户端；新增消息类型应先在客户端做可选处理。
