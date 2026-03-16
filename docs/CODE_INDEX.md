# CODE_INDEX.md - 快闪群聊 App

## 项目结构

```
flowCircle/
├── apps/
│   ├── mobile/                    # 移动端代码
│   │   ├── ios/                   # iOS 原生 App
│   │   │   ├── flowCircle/
│   │   │   │   ├── Views/         # SwiftUI 视图
│   │   │   │   ├── Models/        # 数据模型
│   │   │   │   ├── Services/      # 网络/WS服务
│   │   │   │   └── Utils/         # 工具类
│   │   │   └── flowCircle.xcodeproj
│   │   └── android/               # Android 原生 App
│   │       ├── app/src/main/java/com/flowcircle/
│   │       │   ├── ui/            # Compose UI
│   │       │   ├── data/          # 数据层
│   │       │   ├── network/       # 网络服务
│   │       │   └── utils/         # 工具类
│   │       └── build.gradle
│   └── web/                       # Web/H5 版本 (可选)
├── server/                        # 服务端代码
│   ├── src/
│   │   ├── api/                   # REST API 路由
│   │   │   ├── auth.js            # 认证相关
│   │   │   ├── groups.js          # 群聊API
│   │   │   └── messages.js        # 消息API
│   │   ├── services/              # 业务逻辑
│   │   │   ├── userService.js     # 用户服务
│   │   │   ├── groupService.js    # 群聊服务
│   │   │   ├── messageService.js  # 消息服务
│   │   │   ├── heatService.js     # 热度服务
│   │   │   ├── danmakuService.js  # 弹幕服务
│   │   │   └── botService.js      # 机器人服务
│   │   ├── websocket/             # WebSocket 处理
│   │   │   ├── index.js           # WS服务器
│   │   │   ├── handlers/          # 事件处理器
│   │   │   └── middleware/        # WS中间件
│   │   ├── models/                # 数据模型
│   │   │   ├── user.js            # 用户模型
│   │   │   ├── group.js           # 群组模型
│   │   │   └── message.js         # 消息模型
│   │   ├── utils/                 # 工具函数
│   │   │   ├── validator.js       # 验证器
│   │   │   ├── sensitiveWords.js  # 敏感词过滤
│   │   │   └── logger.js          # 日志
│   │   └── jobs/                  # 定时任务
│   │       └── botScheduler.js    # 机器人任务
│   ├── config/
│   │   ├── database.js            # 数据库配置
│   │   ├── redis.js               # Redis配置
│   │   └── socket.js              # Socket配置
│   └── server.js                  # 入口文件
├── shared/                        # 共享代码
│   ├── constants/                 # 常量定义
│   ├── types/                     # TypeScript类型
│   └── utils/                     # 通用工具
└── scripts/                       # 部署脚本
    ├── deploy.sh
    └── setup.sh
```

## 关键文件速查

### 配置文件

| 文件 | 说明 |
|------|------|
| `server/config/database.js` | PostgreSQL 连接配置 |
| `server/config/redis.js` | Redis 连接配置 |
| `server/.env.example` | 环境变量模板 |

### 核心服务

| 文件 | 功能 |
|------|------|
| `server/src/services/heatService.js` | 热度计算与更新 |
| `server/src/services/danmakuService.js` | 弹幕生成与防重叠 |
| `server/src/services/botService.js` | 系统机器人消息生成 |
| `server/src/websocket/handlers/message.js` | 消息收发处理 |

### 数据模型

| 文件 | 对应表 |
|------|--------|
| `server/src/models/user.js` | users 表 |
| `server/src/models/group.js` | groups 表 |
| `server/src/models/message.js` | Redis 消息结构 |

### 客户端核心

| 文件 (iOS) | 功能 |
|------------|------|
| `apps/mobile/ios/flowCircle/Views/HomeView.swift` | 首页广场 |
| `apps/mobile/ios/flowCircle/Views/GroupChatView.swift` | 群聊房间 |
| `apps/mobile/ios/flowCircle/Services/WebSocketService.swift` | WebSocket连接 |
| `apps/mobile/ios/flowCircle/Views/Components/FloatingCard.swift` | 游走卡片组件 |
| `apps/mobile/ios/flowCircle/Views/Components/BreathingLight.swift` | 呼吸灯效果 |

| 文件 (Android) | 功能 |
|----------------|------|
| `apps/mobile/android/ui/home/HomeScreen.kt` | 首页广场 |
| `apps/mobile/android/ui/chat/ChatScreen.kt` | 群聊房间 |
| `apps/mobile/android/network/WebSocketManager.kt` | WebSocket连接 |

## 快速导航

### 开发新功能

1. **新增 API 接口** → `server/src/api/`
2. **新增业务逻辑** → `server/src/services/`
3. **新增数据库表** → `server/src/models/` + migrations
4. **新增 WebSocket 事件** → `server/src/websocket/handlers/`

### 调试问题

1. **消息收不到** → 检查 `server/src/websocket/handlers/message.js`
2. **热度不更新** → 检查 `server/src/services/heatService.js`
3. **弹幕重叠** → 检查 `server/src/services/danmakuService.js`
4. **敏感词过滤失效** → 检查 `server/src/utils/sensitiveWords.js`

### 性能优化

1. **Redis 查询慢** → 检查 Key 设计和过期策略
2. **WebSocket 连接多** → 考虑分片或集群
3. **首页卡顿** → 优化卡片数量和动画性能

## 命名规范

### 文件命名
- 服务端: `camelCase.js`
- 客户端 (iOS): `PascalCase.swift`
- 客户端 (Android): `PascalCase.kt`

### 变量命名
- 数据库表: 小写 + 下划线 (`user_groups`)
- Redis Key: 冒号分隔 (`group:{id}:messages`)
- API 路由: RESTful (`/api/v1/groups/:id`)

## 环境依赖

### 服务端
- Node.js >= 18
- Redis >= 6
- PostgreSQL >= 14

### 客户端 (iOS)
- Xcode >= 15
- iOS >= 15
- Swift >= 5.9

### 客户端 (Android)
- Android Studio Hedgehog
- minSdk 24
- targetSdk 34

## 常用命令

```bash
# 服务端启动
cd server && npm install && npm run dev

# iOS 启动
cd apps/mobile/ios && open flowCircle.xcodeproj

# Android 启动
cd apps/mobile/android && ./gradlew installDebug

# 数据库迁移
cd server && npm run migrate

# 运行测试
cd server && npm test
cd apps/mobile/ios && xcodebuild test
```
