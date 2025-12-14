# OurChat2.0 完整 MVC 分层与包结构设计

## 一、总体包结构

```
com.ourchat
 ├── controller          # 请求入口（HTTP / WebSocket）
 │    ├── login
 │    │    └── LoginServlet          # 用户登录
 │    ├── logout
 │    │    └── LogoutServlet         # 用户主动登出
 │    ├── user
 │    │    ├── RegisterServlet       # 用户注册
 │    │    └── UserListServlet       # 查询用户列表（在线/离线）
 │    ├── chat
 │    │    ├── ChatHistoryServlet    # 初次进入聊天室加载历史消息
 │    │    └── ChatWebSocket         # WebSocket，处理实时聊天
 │    └── filter
 │         └── AuthFilter            # 拦截未登录请求
 ├── service             # 核心业务逻辑
 │    ├── UserService
 │    ├── MessageService
 │    ├── SystemMessageService
 │    └── OnlineUserService
 ├── dao                 # 数据持久化访问
 │    ├── UserDao
 │    ├── ChatMessageDao
 │    └── SystemMessageDao
 ├── model               # 实体对象
 │    ├── User
 │    ├── ChatMessage
 │    └── SystemMessage
 ├── listener            # 生命周期监听器
 │    ├── SessionListener
 │    └── AppStartupListener
 └── util                # 工具类
      ├── EncryptUtil
      ├── DateTimeUtil
      └── Constants
```

---

## 二、Controller 层（请求入口）

| 类名                   | 类型                 | 职责说明                              |
| -------------------- | ------------------ | --------------------------------- |
| `LoginServlet`       | Servlet            | 校验邮箱/密码，单端登录检查，创建 Session，初始化在线状态 |
| `LogoutServlet`      | Servlet            | 用户主动登出，清理 Session，更新在线状态，生成系统下线消息 |
| `RegisterServlet`    | Servlet            | 用户注册，校验邮箱唯一性并写入数据库                |
| `UserListServlet`    | Servlet            | 返回在线/离线用户列表，供前端初始化或刷新             |
| `ChatHistoryServlet` | Servlet            | 初次进入聊天室加载历史消息（群聊 + 私聊）            |
| `ChatWebSocket`      | WebSocket Endpoint | 实时消息收发，推送系统消息，私聊判断                |
| `AuthFilter`         | Filter             | 拦截未登录请求，禁止访问聊天室 JSP 页面            |

**设计特点**：

* 每个 Servlet 只做一件事，无 `action` 参数
* WebSocket 仅在 Controller 层出现
* 与 Service 层分离，Controller 不做业务判断

---

## 三、Service 层（业务逻辑）

| 类名                     | 职责说明                                     |
| ---------------------- | ---------------------------------------- |
| `UserService`          | 用户注册、登录、登出逻辑，单端登录检查，提供用户信息查询             |
| `MessageService`       | 创建聊天消息，判断消息可见性（群聊/私聊），存入数据库              |
| `SystemMessageService` | 生成系统消息（上线/下线），存入数据库，并推送给在线用户             |
| `OnlineUserService`    | 内存维护在线用户列表、Session/WebSocket 映射，提供在线人数统计 |

---

## 四、DAO 层（数据访问）

| 类名                 | 职责说明    |
| ------------------ | ------- |
| `UserDao`          | 用户表增删改查 |
| `ChatMessageDao`   | 聊天消息表增查 |
| `SystemMessageDao` | 系统消息表增查 |

---

## 五、Model 层（实体对象）

| 类名              | 字段           | 类型        | 来源  | 说明                      |
| --------------- | ------------ | --------- | --- | ----------------------- |
| `User`          | id           | Long      | 数据库 | 用户唯一ID                  |
|                 | nickname     | String    | 数据库 | 昵称                      |
|                 | email        | String    | 数据库 | 邮箱                      |
|                 | password     | String    | 数据库 | 加密密码                    |
|                 | createTime   | Timestamp | 数据库 | 注册时间                    |
|                 | online       | Boolean   | 运行态 | 是否在线                    |
|                 | sessionId    | String    | 运行态 | 当前 Session/WebSocket ID |
| `ChatMessage`   | id           | Long      | 数据库 | 消息唯一ID                  |
|                 | fromUserId   | Long      | 数据库 | 发送人ID                   |
|                 | fromNickname | String    | 数据库 | 发送人昵称（冗余）               |
|                 | toUserId     | Long      | 数据库 | 接收人ID（NULL 表示群聊）        |
|                 | toNickname   | String    | 数据库 | 接收人昵称（冗余）               |
|                 | content      | String    | 数据库 | 消息内容                    |
|                 | type         | Enum      | 数据库 | 消息类型：群聊/私聊              |
|                 | createTime   | Timestamp | 数据库 | 发送时间                    |
| `SystemMessage` | id           | Long      | 数据库 | 系统消息ID                  |
|                 | userId       | Long      | 数据库 | 上下线用户ID                 |
|                 | nickname     | String    | 数据库 | 上下线用户昵称（冗余）             |
|                 | action       | Enum      | 数据库 | 上线/下线动作                 |
|                 | createTime   | Timestamp | 数据库 | 消息时间                    |

---

## 六、Listener 层（生命周期监听器）

| 类名                   | 类型                     | 职责说明                                                                 |
| -------------------- | ---------------------- | -------------------------------------------------------------------- |
| `SessionListener`    | HttpSessionListener    | Session 生命周期监听，用户关闭浏览器或超时触发下线逻辑，调用 `OnlineUserService` 更新在线状态并生成系统消息 |
| `AppStartupListener` | ServletContextListener | 初始化在线用户列表、缓存等运行态数据                                                   |

---

## 七、Util 工具类

| 类名             | 职责说明            |
| -------------- | --------------- |
| `EncryptUtil`  | 密码加密/验证         |
| `DateTimeUtil` | 时间格式化           |
| `Constants`    | 消息类型、系统动作、枚举常量等 |

---

## 八、分层职责总结

| 层          | 职责                                |
| ---------- | --------------------------------- |
| Controller | 请求入口（HTTP / WebSocket），调用 Service |
| Service    | 核心业务逻辑（登录/登出、消息处理、在线用户管理）         |
| DAO        | 数据库访问（CRUD）                       |
| Model      | 数据结构（数据库字段 + 运行态字段）               |
| Listener   | 生命周期监听器，维护运行态信息                   |
| Util       | 工具类（加密、时间、常量）                     |

