# OurChat2.0 WebSocket 消息协议设计

## 一、消息类型概览

| 消息类型      | 方向        | 说明                 |
| --------- | --------- | ------------------ |
| CHAT      | 双向        | 聊天消息（群聊或私聊）        |
| SYSTEM    | 服务器 → 客户端 | 系统消息，如上线/下线通知      |
| USER_LIST | 服务器 → 客户端 | 当前在线/离线用户列表更新      |
| ERROR     | 双向        | 错误提示（如消息发送失败、非法请求） |

---

## 二、消息结构设计

### 1️⃣ CHAT 消息

* **用途**：发送聊天内容（群聊 / 私聊）
* **方向**：客户端 → 服务器、服务器 → 客户端
* **JSON 示例**：

```json
{
  "type": "CHAT",
  "fromUserId": 1,
  "fromNickname": "Alice",
  "toUserId": 2,        // null 表示群聊
  "toNickname": "Bob",  // null 表示群聊
  "content": "Hello, Bob!",
  "messageType": "PRIVATE",  // "PRIVATE" 或 "GROUP"
  "createTime": "2025-12-14T10:30:00"
}
```

* **说明**：

  * `toUserId = null` 表示群聊
  * `messageType` 用于区分群聊 / 私聊
  * 服务器在广播消息时填充 `fromNickname`，客户端直接显示
  * `createTime` 可选，服务器生成保证时间统一

---

### 2️⃣ SYSTEM 消息

* **用途**：上线 / 下线通知
* **方向**：服务器 → 客户端
* **JSON 示例**：

```json
{
  "type": "SYSTEM",
  "userId": 2,
  "nickname": "Bob",
  "action": "ONLINE",   // "ONLINE" 或 "OFFLINE"
  "createTime": "2025-12-14T10:31:00"
}
```

* **说明**：

  * 前端仅显示最新一条系统消息
  * `action` 决定显示“上线”或“下线”
  * 用于动态刷新在线/离线列表

---

### 3️⃣ USER_LIST 消息

* **用途**：更新在线 / 离线用户列表
* **方向**：服务器 → 客户端
* **JSON 示例**：

```json
{
  "type": "USER_LIST",
  "onlineUsers": [
    {"userId": 1, "nickname": "Alice"},
    {"userId": 3, "nickname": "Charlie"}
  ],
  "offlineUsers": [
    {"userId": 2, "nickname": "Bob"}
  ],
  "onlineCount": 2
}
```

* **说明**：

  * `onlineUsers` 与 `offlineUsers` 数组用于前端渲染用户列表
  * `onlineCount` 用于显示在线人数
  * 服务器在用户上下线或周期刷新时推送

---

### 4️⃣ ERROR 消息

* **用途**：客户端或服务器通知错误
* **方向**：双向
* **JSON 示例**：

```json
{
  "type": "ERROR",
  "code": 1001,
  "message": "用户不存在或已离线"
}
```

* **说明**：

  * `code` 为错误编号，便于客户端处理
  * `message` 为人类可读的提示信息

---

## 三、消息协议总结

| 消息类型      | 必要字段                                   | 可选字段                                           | 说明          |
| --------- | -------------------------------------- | ---------------------------------------------- | ----------- |
| CHAT      | type, fromUserId, content, messageType | toUserId, toNickname, fromNickname, createTime | 群聊 / 私聊消息   |
| SYSTEM    | type, userId, action                   | nickname, createTime                           | 上线/下线通知     |
| USER_LIST | type, onlineUsers, offlineUsers        | onlineCount                                    | 在线/离线用户列表刷新 |
| ERROR     | type, code, message                    | -                                              | 错误提示消息      |

