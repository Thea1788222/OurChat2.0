# OurChat2.0 关键业务流程时序设计

---

## 1️⃣ 用户登录 → 进入聊天室

1. 用户在登录页输入邮箱和密码，提交表单。
2. 表单请求被 `LoginServlet` 接收。
3. `LoginServlet` 调用 `UserService` 校验邮箱和密码：

   * 查询数据库中用户信息（`UserDao`）。
   * 验证密码。
   * 检查是否已有有效 Session（单端登录）。
4. 登录成功：

   * 创建新的 Session。
   * 将用户标记为在线（`OnlineUserService` 更新 `User.online` 和 `sessionId`）。
5. 系统生成上线消息（`SystemMessageService`）：

   * 保存到数据库（`SystemMessageDao`）。
6. `LoginServlet` 返回聊天室页面 JSP。
7. 前端 JSP 建立 WebSocket 连接 (`ChatWebSocket`)。
8. WebSocket `onOpen` 回调触发：

   * Service 层更新 WebSocket 映射。
   * 广播系统上线消息给其他在线用户。
   * 在线用户列表刷新。

**结果**：用户登录成功，在线状态更新，系统消息推送完成，前端聊天室界面显示历史消息与在线用户列表。

---

## 2️⃣ 用户发送消息（群聊 / 私聊）

1. 用户在聊天室输入消息内容，并选择可见对象（全员 / 单个用户）。
2. 点击发送按钮，前端通过 WebSocket 发送消息数据（发送人 ID、消息内容、接收人 ID）。
3. WebSocket Endpoint (`ChatWebSocket`) 接收消息：

   * 调用 `MessageService` 创建 `ChatMessage` 实体。
   * 判断消息类型：

     * `toUserId = null` → 群聊。
     * `toUserId != null` → 私聊。
4. `MessageService` 将消息写入数据库 (`ChatMessageDao`)。
5. `OnlineUserService` 查询在线接收用户。
6. WebSocket 推送消息：

   * 在线用户立即接收。
   * 离线用户消息仅存数据库。
7. 前端接收消息：

   * 动态渲染聊天区。
   * 滚动到最新消息。
   * 昵称颜色根据在线状态和是否“我”显示（红/绿/灰）。

**结果**：消息成功发送，数据库记录持久化，在线用户实时接收，前端动态更新。

---

## 3️⃣ 用户下线（三种方式）

### A. 用户主动点击“退出登录”

1. 用户点击“退出登录”按钮。
2. 请求被 `LogoutServlet` 接收。
3. `LogoutServlet` 调用 `UserService` 清理 Session，并通过 `OnlineUserService` 更新在线状态。
4. `SystemMessageService` 生成系统下线消息，写入数据库。
5. WebSocket 广播系统下线消息给其他在线用户。
6. 前端收到消息，更新在线/离线列表和系统提示区。
7. 用户被重定向到登录页。

---

### B. 用户直接关闭浏览器或标签页

1. 浏览器关闭触发 Session 超时或 WebSocket `onClose`。
2. `SessionListener` 或 `ChatWebSocket onClose` 调用 `OnlineUserService` 更新用户 `online = false`。
3. `SystemMessageService` 生成系统下线消息并存入数据库。
4. WebSocket 广播给其他在线用户。
5. 前端更新用户列表和系统提示区。

> 注意：可允许一定延时，不要求毫秒级实时。

---

### C. Session 超时（用户长时间未操作）

1. 容器触发 Session 过期事件。
2. `SessionListener` 捕获 `sessionDestroyed`。
3. 更新在线状态（`OnlineUserService`）。
4. 系统生成下线消息（`SystemMessageService`）并广播。
5. 在线用户前端刷新显示。

---

## 总结

* **登录流程**：前端表单 → Servlet → Service → DAO → Listener / WebSocket → 前端
* **发送消息**：前端 WebSocket → ChatWebSocket → Service → DAO → OnlineUserService → WebSocket → 前端
* **下线流程**：主动登出 / 浏览器关闭 / Session 超时 → Servlet / Listener → Service → DAO → WebSocket → 前端
