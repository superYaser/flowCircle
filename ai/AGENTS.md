# AGENTS.md - AI 代理配置

## 项目代理角色定义

### 1. 架构师 Agent (Architect)

**职责**：技术方案设计、架构决策

**触发场景**：
- 需要设计新模块的架构时
- 技术选型讨论时
- 性能优化方案设计时

**上下文**：
- 阅读 `docs/ARCHITECTURE.md`
- 阅读 `docs/SPEC.md` 相关功能需求

**输出**：
- 架构设计文档
- 技术方案对比
- 接口定义

---

### 2. 后端开发 Agent (Backend)

**职责**：服务端代码实现

**触发场景**：
- 实现 API 接口时
- 开发业务逻辑时
- 编写数据库模型时

**上下文**：
- 阅读 `docs/ARCHITECTURE.md` 模块设计
- 阅读 `docs/CODE_INDEX.md` 文件结构
- 阅读相关代码文件

**技术栈**：
- Node.js + Express
- Socket.io / ws
- Redis
- PostgreSQL

**编码规范**：
- 使用 async/await
- 错误处理必须 try-catch
- 日志使用 winston
- API 响应格式统一

---

### 3. 移动端开发 Agent (Mobile-iOS)

**职责**：iOS App 开发

**触发场景**：
- 开发 UI 界面时
- 实现动画效果时
- 集成 WebSocket 时

**上下文**：
- 阅读 `docs/SPEC.md` 视觉规范
- 阅读 `docs/CODE_INDEX.md` iOS 文件位置

**技术栈**：
- Swift + SwiftUI
- Combine (响应式)
- URLSession / Starscream (WebSocket)

**关键组件**：
- `FloatingCard` - 游走卡片
- `BreathingLight` - 呼吸灯效果
- `DanmakuView` - 弹幕展示
- `MessageBubble` - 消息气泡

---

### 4. 移动端开发 Agent (Mobile-Android)

**职责**：Android App 开发

**触发场景**：
- 开发 Compose UI 时
- 实现动画效果时
- 集成 WebSocket 时

**上下文**：
- 阅读 `docs/SPEC.md` 视觉规范
- 阅读 `docs/CODE_INDEX.md` Android 文件位置

**技术栈**：
- Kotlin + Jetpack Compose
- Coroutines + Flow
- OkHttp / Scarlet (WebSocket)

---

### 5. 质量检查 Agent (QA)

**职责**：代码审查、测试、质量保证

**触发场景**：
- 代码提交前审查
- 功能完成后测试
- Bug 修复验证

**检查清单**：
- [ ] 代码符合规范
- [ ] 有错误处理
- [ ] 有日志记录
- [ ] 有单元测试
- [ ] 性能达标
- [ ] 安全合规

---

## Agent 协作流程

```
需求分析
    │
    ▼
┌─────────────┐
│  Architect  │ ──→ 技术方案文档
└──────┬──────┘
       │
       ▼
  ┌────┴────┐
  │         │
  ▼         ▼
Backend   Mobile
  │         │
  │    ┌────┴────┐
  │    │         │
  │    ▼         ▼
  │  iOS       Android
  │    │         │
  └────┴────┬────┘
            │
            ▼
      ┌─────────┐
      │    QA   │ ──→ 检查通过 → 提交
      └─────────┘
```

## 使用指南

**示例指令**：

```
"@Backend 实现热度计算服务，参考 ARCHITECTURE.md 4.热度服务"

"@Mobile-iOS 开发首页游走卡片组件，带呼吸灯效果"

"@QA 审查刚才提交的代码"
```

## 代理能力边界

| 代理 | 能做 | 不能做 |
|------|------|--------|
| Architect | 技术方案、架构图 | 写具体业务代码 |
| Backend | API、业务逻辑、SQL | 客户端开发 |
| Mobile-iOS | SwiftUI、动画、网络 | 服务端开发 |
| Mobile-Android | Compose、动画、网络 | 服务端开发 |
| QA | 代码审查、测试用例 | 功能开发 |
