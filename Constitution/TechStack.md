# HireFlow Tech Stack

> 状态：基线（v1.0）  
> 最后更新：2026-10-28  
> 维护规则：技术变更必须先评估影响，确认后同步更新本文和 `DECISIONS.md`。

## 技术基线

| 层次 | 选择 | 用途与约束 |
| --- | --- | --- |
| 语言 | Java 21 LTS（Homebrew Temurin） | 后端主语言；本机已安装 `openjdk@21` |
| 构建 | Maven 3.9.x | 后端依赖和生命周期管理 |
| 后端框架 | Spring Boot 3.4.x | REST API、配置和测试基础 |
| Web | Spring MVC + Bean Validation | 控制器、参数校验和异常处理 |
| 鉴权 | JWT + Spring MVC Interceptor + BCrypt | 不引入 Spring Security 过滤器链 |
| 持久层 | MyBatis-Plus 3.5.x | 基础 CRUD，关键查询手写 SQL |
| 数据库 | MySQL 8.4 | 本地开发库 `hireflow`，测试库 `hireflow_test` |
| 接口测试 | JUnit 5 + MockMvc（暂缓） | 工具保留；当前开发阶段不新增自动化测试 |
| 前端 | Vue 3 + TypeScript + Vite | 页面、路由和组件开发 |
| UI | Element Plus | 表单、表格、分页、弹窗和状态提示 |
| 前端状态 | Pinia + Vue Router | 登录状态、角色路由和页面切换 |
| 请求 | Axios | 统一请求封装和响应拦截器 |
| 文档 | Markdown | 接口、错误码、学习笔记和面试材料 |

## 工程约束

- 三个独立 Git 仓库：`backend`、`frontend`、`docs`；外层 `HireFlow/` 不是 Git 仓库。
- 项目名统一为 `HireFlow`，Java 包名为 `com.hireflow`，数据库名为 `hireflow`。
- 依赖版本在初始化时锁定，提交 `pom.xml`、`package.json` 和 `package-lock.json`。
- 数据库账号、密码和 JWT 密钥通过环境变量配置，不提交真实密钥。
- 使用 `schema.sql` 和 `data.sql` 手动初始化数据库，不使用自动建表。
- 使用本地 MySQL 和本机前后端进程开发，不安装 Docker。
- 接口统一返回 `{ code, message, data }`，HTTP 状态码采用标准映射。
- 分页参数从 1 开始，默认每页 10 条，最大 50 条。
- JWT 有效期 2 小时，前端保存在 `localStorage`，请求使用 `Authorization: Bearer`。
- 前端使用 Axios 拦截器统一处理 401 跳转和其他错误提示。

## 明确不引入

- Spring Security 过滤器链、Redis、消息队列、Elasticsearch。
- 文件上传、在线聊天、通知中心、复杂推荐算法。
- Docker、CI/CD、生产级监控和微服务拆分。

## 环境与降级规则

当前机器已安装 Maven、Node.js、npm、MySQL 8.4 和 OpenJDK 21，但 MySQL 服务需要启动，`gh` 尚未安装。

如果环境准备严重阻塞：

- 可以临时使用 Java 17 或 H2 验证思路，但不能静默改变项目基线。
- 任何降级必须同步修改本文和 `DECISIONS.md`，并说明恢复条件。
- 不以“教程默认版本”替代已确认的版本策略。
