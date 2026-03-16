# ARCHITECTURE.md - 快闪群聊 App

## 技术架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                          客户端层                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   iOS App    │  │ Android App  │  │   Web H5     │           │
│  │  (Swift/Kotlin) │  │  (Kotlin)    │  │  (React/Vue) │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
└─────────┼──────────────────┼──────────────────┼───────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │ WebSocket / HTTP
┌────────────────────────────┼─────────────────────────────────────┐
│                      接入层 │                                     │
│  ┌─────────────────────────┴────────────────────────────┐         │
│  │              API Gateway / Load Balancer             │         │
│  └─────────────────────────┬────────────────────────────┘         │
└────────────────────────────┼───────────────────────────────────────┘
                             │
┌────────────────────────────┼───────────────────────────────────────┐
│                      服务层 │                                       │
│  ┌─────────────┐  ┌────────┴────────┐  ┌─────────────────┐         │
│  │   用户服务   │  │    群聊服务      │  │    消息服务      │         │
│  │  User Svc   │  │    Group Svc    │  │   Message Svc   │         │
│  └─────────────┘  └─────────────────┘  └─────────────────┘         │
│  ┌─────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │   热度服务   │  │    弹幕服务      │  │    系统机器人    │         │
│  │  Heat Svc   │  │   Danmaku Svc   │  │      Bot Svc    │         │
│  └─────────────┘  └─────────────────┘  └─────────────────┘         │
│  ┌─────────────┐  ┌─────────────────┐                               │
│  │  审核服务    │  │   推送服务       │                               │
│  │ Moderation  │  │    Push Svc     │                               │
│  └─────────────┘  └─────────────────┘                               │
└──────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┼───────────────────────────────────────┐
│                      数据层 │                                       │
│  ┌─────────────┐  ┌────────┴────────┐  ┌─────────────────┐         │
│  │    Redis    │  │   PostgreSQL    │  │   Elasticsearch │         │
│  │  (消息缓存  │  │  (持久化数据)    │  │  (日志/分析)    │         │
│  │   热度计算) │  │                  │  │                  │         │
│  └─────────────┘  └─────────────────┘  └─────────────────┘         │
└──────────────────────────────────────────────────────────────────────┘
```

## 技术栈选型

### 客户端
| 平台 | 技术 | 说明 |
|------|------|------|
| iOS | Swift + SwiftUI | 原生开发，动画性能最佳 |
| Android | Kotlin + Jetpack Compose | 原生开发，与iOS对齐 |
| 跨端备选 | Flutter | 如需快速迭代可考虑 |

### 服务端
| 组件 | 技术 | 说明 |
|------|------|------|
| 网关 | Nginx + Node.js | WebSocket代理，负载均衡 |
| API服务 | Node.js + Express | RESTful API |
| 实时通信 | Socket.io / ws | WebSocket连接管理 |
| 任务调度 | Bull + Redis | 定时任务队列 |

### 数据存储
| 类型 | 技术 | 用途 |
|------|------|------|
| 缓存 | Redis | 消息队列(5分钟过期)、热度计算、在线状态 |
| 数据库 | PostgreSQL | 用户信息、群信息、关系数据 |
| 搜索 | Elasticsearch | 消息搜索、日志分析 |

## 核心模块设计

### 1. 用户模块 (User Service)

```
用户表 (users)
├── id: UUID (PK)
├── phone: String (唯一)
├── nickname: String
├── avatar: String (URL)
├── created_at: Timestamp
└── last_active: Timestamp

用户群关系表 (user_groups)
├── user_id: UUID (FK)
├── group_id: UUID (FK)
├── role: Enum [member, owner]
├── joined_at: Timestamp
└── PK: (user_id, group_id)
```

**限制规则**：
- 一个用户只能创建一个群（role=owner）
- 最多加入20个群
- 解散群后4小时冷却期

### 2. 群聊模块 (Group Service)

```
群组表 (groups)
├── id: UUID (PK)
├── name: String (最多8个汉字)
├── owner_id: UUID (FK → users)
├── type: Enum [system, user]
├── category: String [徒步,钓鱼,股票,麻将,辅导小孩]
├── member_count: Integer (0-20)
├── heat_level: Enum [low, medium, high]
├── heat_score: Integer (最近1分钟消息数)
├── created_at: Timestamp
└── dissolved_at: Timestamp (nullable)
```

**系统群初始化**：
```sql
INSERT INTO groups (name, type, category) VALUES
('徒步', 'system', 'outdoor'),
('钓鱼', 'system', 'hobby'),
('股票', 'system', 'finance'),
('麻将', 'system', 'game'),
('辅导小孩', 'system', 'education');
```

### 3. 消息模块 (Message Service)

```
消息结构 (Redis Hash)
Key: group:{group_id}:messages
├── id: UUID
├── group_id: UUID
├── sender_id: UUID
├── sender_name: String
├── content: String (最多50字)
├── is_truncated: Boolean (是否超过24字)
├── created_at: Timestamp
└── expires_at: Timestamp (created_at + 5分钟)

Redis策略：
- 使用Sorted Set，score为timestamp
- 过期时间：5分钟
- 自动清理：Redis TTL + 定时任务兜底
```

### 4. 热度服务 (Heat Service)

```
热度计算算法：

1. 数据收集
   - 监听消息发布事件
   - 每收到消息，group_id对应计数器+1
   - 计数器窗口：滑动窗口1分钟

2. 热度分级
   low:    count < 3
   medium: 3 <= count < 10
   high:   10 <= count < 30
   extreme: count >= 30

3. 呼吸灯效果映射
   low:    border-color: #4A90E2, animation-duration: 3s
   medium: border-color: #F5A623, animation-duration: 1.5s
   high:   border-color: #D0021B, animation-duration: 0.8s, scale: 1.05

4. 数据刷新
   - WebSocket推送热度变更
   - 客户端每10秒轮询作为兜底
```

### 5. 弹幕服务 (Danmaku Service)

```
弹幕结构：
{
  id: UUID,
  content: String,
  trajectory: Integer (0-4, 轨迹编号),
  created_at: Timestamp,
  expires_at: Timestamp (3分钟)
}

防重叠算法：
1. 首页分为5条轨迹（Y轴均匀分布）
2. 每条轨迹记录最后弹幕的X坐标
3. 新弹幕优先选择X坐标最小的轨迹
4. 同轨迹弹幕间隔至少200px
```

### 6. 系统机器人 (Bot Service)

```
定时任务配置：

Job: generate_bot_message
├── cron: */5 * * * * (每5分钟)
├── groups: [徒步群,钓鱼群,股票群,麻将群,辅导小孩群]
└── action: 
    1. 搜索对应关键词的最新资讯
    2. 生成摘要（12字×2行）
    3. 以系统身份发送到对应群

内容源：
- 徒步: 户外资讯API / 小红书热点
- 钓鱼: 垂钓资讯API
- 股票: 财经API (如东方财富)
- 麻将: 棋牌资讯
- 辅导: 教育资讯API
```

## WebSocket 事件设计

### 客户端 → 服务端

| 事件 | 参数 | 说明 |
|------|------|------|
| join_group | {group_id} | 加入群聊房间 |
| leave_group | {group_id} | 离开群聊房间 |
| send_message | {group_id, content} | 发送消息 |
| send_danmaku | {content} | 发送弹幕 |
| delete_message | {group_id, message_id} | 删除消息(群主) |

### 服务端 → 客户端

| 事件 | 参数 | 说明 |
|------|------|------|
| message_new | {message} | 新消息通知 |
| message_deleted | {message_id} | 消息被删除 |
| heat_updated | {group_id, level, score} | 热度更新 |
| danmaku_new | {danmaku} | 新弹幕 |
| member_joined | {user_id} | 成员加入 |
| member_left | {user_id} | 成员离开 |

## API 接口设计

### 用户相关

```
POST   /api/v1/auth/verify-code     # 发送验证码
POST   /api/v1/auth/login           # 登录/注册
GET    /api/v1/user/profile         # 获取用户信息
PUT    /api/v1/user/profile         # 更新用户信息
```

### 群聊相关

```
GET    /api/v1/groups               # 获取群列表（首页游走卡片）
POST   /api/v1/groups               # 创建群
GET    /api/v1/groups/:id           # 获取群详情
PUT    /api/v1/groups/:id           # 修改群信息
DELETE /api/v1/groups/:id           # 解散群
POST   /api/v1/groups/:id/join      # 加入群
POST   /api/v1/groups/:id/leave     # 离开群
```

### 消息相关

```
GET    /api/v1/groups/:id/messages  # 获取历史消息（最近5分钟）
GET    /api/v1/danmaku              # 获取当前弹幕列表
```

## 安全设计

### 1. 关键词过滤

```
流程：
1. 客户端输入时本地校验（轻量过滤）
2. 服务端接收时严格校验
3. 触发敏感词：返回成功但标记为blocked，不广播

实现：
- 敏感词库：Redis Set存储
- 匹配算法：DFA算法，O(1)匹配
- 更新机制：热更新，无需重启服务
```

### 2. 频率限制

| 操作 | 限制 |
|------|------|
| 发送消息 | 10条/分钟 |
| 发送弹幕 | 5条/分钟 |
| 创建群 | 1个/4小时 |
| 加入群 | 20个上限 |

## 部署架构

```
                          ┌─────────────┐
                          │   CDN       │
                          │ (静态资源)   │
                          └──────┬──────┘
                                 │
                          ┌──────▼──────┐
                          │  Cloudflare │
                          │   (WAF)     │
                          └──────┬──────┘
                                 │
┌────────────────────────────────┼────────────────────────────────┐
│                         Kubernetes Cluster                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ API Pod  │  │ API Pod  │  │ WS Pod   │  │ WS Pod   │       │
│  │   x2     │  │   x2     │  │   x2     │  │   x2     │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ Bot Pod  │  │ Bot Pod  │  │ Worker   │                     │
│  │   x1     │  │   x1     │  │   x2     │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
┌──────▼──────┐  ┌────────▼────────┐  ┌──────▼──────┐
│   Redis     │  │   PostgreSQL    │  │ Elasticsearch│
│   Cluster   │  │    Primary      │  │    Cluster   │
└─────────────┘  │    + Replica    │  └─────────────┘
                 └─────────────────┘
```

## 性能指标

| 指标 | 目标 |
|------|------|
| 消息投递延迟 | < 100ms |
| 热度计算延迟 | < 50ms |
| 弹幕渲染帧率 | > 30fps |
| 首页加载时间 | < 1s |
| 同时在线用户 | 10,000+ |
| QPS | 10,000+ |
