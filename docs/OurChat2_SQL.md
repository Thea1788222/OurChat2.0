# 1. User 表

```sql
CREATE TABLE `user` (
    `id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '用户唯一ID',
    `nickname` VARCHAR(50) NOT NULL COMMENT '用户昵称',
    `email` VARCHAR(100) NOT NULL UNIQUE COMMENT '用户邮箱（唯一，可登录）',
    `password` VARCHAR(255) NOT NULL COMMENT '加密后的密码',
    `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '注册时间'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

**说明：**

* `id` 为主键，自动生成
* `email` 设置唯一约束
* 不存储在线状态和 Session 信息（运行态字段内存维护）
* `nickname` 可重复

---

# 2. ChatMessage 表

```sql
CREATE TABLE `chat_message` (
    `id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '聊天消息唯一ID',
    `from_user_id` BIGINT NOT NULL COMMENT '发送者用户ID',
    `from_nickname` VARCHAR(50) NOT NULL COMMENT '发送者昵称（冗余）',
    `to_user_id` BIGINT DEFAULT NULL COMMENT '接收者用户ID，NULL表示全员可见',
    `to_nickname` VARCHAR(50) DEFAULT NULL COMMENT '接收者昵称（冗余，私聊时有值）',
    `content` TEXT NOT NULL COMMENT '消息内容',
    `type` ENUM('GROUP','PRIVATE') NOT NULL COMMENT '消息类型：群聊或私聊',
    `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '发送时间',
    CONSTRAINT `fk_chat_from_user` FOREIGN KEY (`from_user_id`) REFERENCES `user`(`id`) ON DELETE CASCADE,
    CONSTRAINT `fk_chat_to_user` FOREIGN KEY (`to_user_id`) REFERENCES `user`(`id`) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='聊天消息表';
```

**说明：**

* `from_user_id` 外键关联 User 表，发送者必须存在
* `to_user_id` 可为空，表示群聊
* `from_nickname` / `to_nickname` 冗余字段，便于显示
* `type` 区分群聊和私聊
* 消息内容使用 TEXT，前端可限制最大字数

---

# 3. SystemMessage 表

```sql
CREATE TABLE `system_message` (
    `id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '系统消息唯一ID',
    `user_id` BIGINT NOT NULL COMMENT '上下线用户ID',
    `nickname` VARCHAR(50) NOT NULL COMMENT '上下线用户昵称（冗余）',
    `action` ENUM('ONLINE','OFFLINE') NOT NULL COMMENT '上下线动作',
    `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '消息生成时间',
    CONSTRAINT `fk_system_user` FOREIGN KEY (`user_id`) REFERENCES `user`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统消息表';
```

**说明：**

* `user_id` 外键关联 User 表
* `action` 用枚举区分上线/下线
* `nickname` 冗余存储，便于快速前端显示
* 前端只显示最新一条系统消息，但历史消息保存在数据库

---

# 4. 总结

* **三张表覆盖所有核心数据**

  * `user`：用户基本信息
  * `chat_message`：聊天记录（群聊 + 私聊）
  * `system_message`：上下线系统消息
* **运行态字段**（在线状态、Session / WebSocket 连接）不存数据库
* 外键设置保证数据完整性
* ENUM 类型用于区分消息类型 / 系统动作
* 数据库设计清晰，后续 DAO / Service / WebSocket 消息结构都能直接使用
