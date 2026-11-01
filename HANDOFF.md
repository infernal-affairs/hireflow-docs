# HireFlow 会话交接

> 核对日期：2026-11-02，Asia/Shanghai。
> 用途：更换模型或供应商后，让新会话从本地文件继续带练。
> 当前阶段：M1 数据库与基础工程，统一响应、异常处理和基础手工验证已完成；下一步处理通用 HTTP 404/405 边界并完成 M1 收尾。

本文记录交接时的事实与既有共识，不替代 Constitution。接手时重新检查文件和 Git 状态；不要假设本文生成后项目没有变化。

## 1. 项目入口与阅读顺序

项目根目录是 `/Users/wujiadai/Documents/ChatGPT/HireFlow/`，外层不是 Git 仓库：

```text
/Users/wujiadai/Documents/ChatGPT/HireFlow/
├── README.md          本地导航
├── TASKS.md           本地任务清单
├── backend/           独立 Git 仓库，GitHub 名称 hireflow-backend
├── frontend/          独立 Git 仓库，GitHub 名称 hireflow-frontend
└── docs/              独立 Git 仓库，GitHub 名称 hireflow-docs
    ├── HANDOFF.md     本交接文档
    └── Constitution/ 项目共识
```

旧目录 `/Users/wujiadai/Documents/ChatGPT/codeForTest/` 已删除。工具若仍携带这个旧工作目录，执行命令时应显式指定现有的项目目录，不能因此重新初始化项目。

先读以下四份宪章，再查看当前代码：

1. [Mission.md](./Constitution/Mission.md)：项目目的、成功标准与带练协议。
2. [TechStack.md](./Constitution/TechStack.md)：技术基线、分层和依赖约束。
3. [RoadMap.md](./Constitution/RoadMap.md)：里程碑和验收条件。
4. [DECISIONS.md](./Constitution/DECISIONS.md)：追加式关键决策记录。

随后按需读取 [M1 学习笔记](./notes/M1-database-and-response-basics.md)、[数据库设计手册](./guides/database-design-playbook.md) 和根目录的本地 `TASKS.md`。根目录文件不在三个子仓库的版本管理中；仅克隆 docs 仓库时未必存在。

## 2. 学习目标与带练规则

学习者是计算机科学与技术应届生，准备投递杭州中小公司的 Java 全栈岗位。最初约有 20 天准备时间；实际剩余时间尚未重新确认，不能把它当作交接日起重新计时的 20 天。

目标是亲手走过关键的全栈开发流程，将理论转换成实践，并能应对面试追问。优先完成可运行的业务闭环，不要求把所有功能一次做完。

必须遵守：

- 使用中文，分小步推进。学习者在本地 JetBrains IntelliJ IDEA 中编写主要业务代码。
- 每项工作开始前说明目标、验收标准、宪章依据，以及是否需要新增依赖。需要依赖时说明名称、作用、版本来源和引入时机。
- 解释“怎样从业务要求分析出这个设计”，不能只给实现结果。
- 新概念首次出现时可以给完整示例；关键语句前加简短注释，示例后解释原因、边界和常见错误。
- 同类任务已示范过后，改为分析框架和提示，由学习者仿写，助手审查。不要直接代写整个业务模块，也不要擅自修改学习者的代码。
- 循环采用“示例讲解 → 学习者仿写 → 审查 → 运行验证 → Git 提交 → 面试复盘”。学习者可以随时求助。
- 当前整个开发阶段暂缓自动化测试，保留已有测试依赖；用编译、启动和手工接口验证推进。不能把“未执行测试”写成“测试通过”。
- 阶段完成要沉淀学习笔记和面试问答。更新文档共识时不能顺便重写用户代码。
- 需要学习者选择方案时，优先使用当前环境可用的交互选择工具，说明各选项利弊；工具不可用时直接说明并用文字选择。
- 关键技术或范围变更先说明影响并取得确认，再更新 Constitution 与决策日志。普通实现步骤不需要反复询问是否允许继续。

## 3. 已确认的技术与业务边界

| 项目 | 当前基线或文件中的版本 | 说明 |
| --- | --- | --- |
| Java | 21；本机为 Homebrew OpenJDK 21.0.7 | 通过 jenv 管理多版本 |
| Maven | 3.9.x；本机为 3.9.14 | Maven 自身版本与运行它的 Java 版本分开看 |
| Spring Boot | `pom.xml` 为 3.4.5 | 不为迁移会话升级依赖 |
| MyBatis-Plus | `pom.xml` 为 3.5.12 | 已引入，业务 Mapper 尚未实现 |
| MySQL | 8.4 | 开发库 `hireflow`、预留测试库 `hireflow_test` |
| Lombok | Spring Boot 父 POM 管理，宪章记录为 1.18.38 | 不再单独写一份版本号 |
| JWT | JJWT 0.12.6 | 依赖已引入，登录鉴权尚未实现 |
| 前端 | Vue 3、TypeScript、Vite、Element Plus、Pinia、Vue Router、Axios | 具体依赖见 package.json 与 package-lock.json |

后端遵循 Controller → Service → Mapper；事务和业务规则放在 Service，Controller 不直接调用 Mapper。Entity、DTO、VO 分离。接口文档使用 Markdown，不引入 Swagger。

鉴权规划为 JWT + Spring MVC Interceptor + BCrypt，不使用 Spring Security 过滤器链。不引入 Docker、Redis、消息队列、微服务或 CI/CD；生产部署和完整前端测试不阻塞当前目标。

数据库账号密码由学习者本人管理，需要时由本人登录；不要索取、读取或记录真实密码。数据库凭据和 JWT 密钥通过环境变量配置，不提交真实凭据。不要假设放入 `.env` 就已经被 Spring Boot 自动加载，应以实际运行配置为准。

已确认但尚未全部实现的业务规则：

- 角色为 `CANDIDATE`、`RECRUITER`；核心流程是发布职位 → 完善资料并投递 → 招聘方推进状态 → 求职者查看进度。
- 投递状态为 `PENDING → INTERVIEWING → OFFERED`；非终态可转 `REJECTED`，`OFFERED`、`REJECTED` 是终态。
- 同一求职者对同一职位永久只能投递一次，由数据库唯一约束兜底。
- 职位下架不删除；下架后详情可见，但不能投递，公开列表不展示。
- 公司归一化名称全局唯一；加入已有公司前必须确认。管理员只管理公司成员，职位权限按发布者校验。
- 分页从 1 开始，默认 10 条、最大 50 条。JWT 有效期 2 小时，前端使用 localStorage 和 `Authorization: Bearer`；Axios 统一处理 401 与错误提示。

这些是后续实现的约束，不能把“已选技术或规则”误报成“功能已完成”。

## 4. 实际完成情况

| 范围 | 交接时状态 | 验证边界 |
| --- | --- | --- |
| M0 | 三仓库、工程骨架、GitHub 初始化已完成 | 原任务记录中有编译、启动、健康接口和前端构建验证；本次未重新执行全部 M0 验收 |
| 数据库 | `users`、`candidate_profiles`、`companies` 的建表 SQL 已有；学习者此前已确认建库建表 | 本次检查了 schema.sql，没有连接 MySQL 读取实时结构 |
| `ApiResponse<T>` | success 与 failure 工厂方法已写好 | 已用于健康接口和异常响应；手工验证通过 |
| `ErrorCode` | 六个错误类型、默认消息和 HttpStatus 已实现 | 已提交；保留显式构造方法，使用 Lombok `@Getter` |
| `BusinessException` | 两个有参构造方法和 getter 已完成 | 2026-11-02 已完成提交前验证，见后端提交 |
| `GlobalExceptionHandler` | 已处理业务异常、字段校验、非法 JSON 和未知异常 | 2026-11-02 手工验证六类主场景；未知异常有服务端堆栈且响应不泄露内部消息 |
| 健康接口 | `/api/v1/health` 已返回 `ApiResponse<Map<String, String>>` | 手工验证 HTTP 200 和统一 JSON 结构通过 |
| 前端业务页面 | 尚未实现 | 已有工程骨架，不代表注册、登录页面完成 |
| M2–M7 | 尚未开始对应业务功能 | 不得直接跳过 M1 验收并宣布完成 |

后端关键文件均位于 `/Users/wujiadai/Documents/ChatGPT/HireFlow/backend/`：

- `pom.xml`
- `sql/schema.sql`、`sql/data.sql`：data.sql 当前为空，尚未准备演示数据。
- `src/main/java/com/hireflow/common/response/ApiResponse.java`
- `src/main/java/com/hireflow/common/exception/ErrorCode.java`
- `src/main/java/com/hireflow/common/exception/BusinessException.java`
- `src/main/java/com/hireflow/health/HealthController.java`

数据库设计要点：`users.email` 非空且唯一；`candidate_profiles.user_id` 兼任主键和引用 `users.id` 的外键；`companies.normalized_name` 唯一，`display_name` 不唯一，`created_by` 引用 `users.id`。尚无 `company_members`、`jobs`、`applications` 表。

学习者已选择：在数据库配置及默认引擎符合要求时，后续表可省略重复表级选项。当前 users 表显式写了引擎和字符集，其余两表省略；不要仅为统一格式要求重建或擅改 SQL。

## 5. 响应与异常设计的接续点

`ApiResponse<T>` 使用 `@Getter`、`@AllArgsConstructor`、`@JsonInclude(ALWAYS)`；字段为 `private final int code`、`String message`、`T data`。

| 已有方法 | 当前语义 |
| --- | --- |
| `success(T data)` | 返回 `ApiResponse<T>`，业务码 0、消息 success、携带数据 |
| `success()` | 返回 `ApiResponse<Void>`，业务码 0、消息 success、data 为 null |
| `failure(ErrorCode errorCode)` | 返回 `ApiResponse<Void>`，使用默认错误消息 |
| `failure(ErrorCode errorCode, String msg)` | 返回 `ApiResponse<Void>`，使用本次自定义消息 |

JSON 固定为 `{ code, message, data }`；无业务数据和失败时 data 为 null，查询得到空列表时 data 是空列表。业务成功码已经确定为 **0**，不要沿用早期讨论中的 200。HTTP 状态码与 JSON 中的业务码是两回事。

| ErrorCode | 业务码 | HTTP 状态 | 默认消息 |
| --- | --- | --- | --- |
| INVALID_PARAM | 1001 | 400 | 参数校验失败 |
| UNAUTHORIZED | 1002 | 401 | 未登录或过期 |
| FORBIDDEN | 1003 | 403 | 无权限执行此操作 |
| NOT_FOUND | 1004 | 404 | 请求的资源不存在 |
| CONFLICT | 1005 | 409 | 数据冲突 |
| INTERNAL_ERROR | 5000 | 500 | 服务器内部错误 |

`BusinessException` 继承 RuntimeException，保存 `private final ErrorCode errorCode`，已有：

- 单参数构造：`super(errorCode.getMessage())`，再保存 errorCode。
- 双参数构造：`super(message)`，再保存 errorCode。
- 手写 `getErrorCode()`；通过继承得到的 `getMessage()` 读取最终异常消息。
- 没有无参构造方法，也没有额外声明 message 字段。

已经建立的调用链是：

```text
Service 判断业务失败并抛出 BusinessException
    ↓
Spring MVC 的全局异常处理器捕获
    ↓
从 getErrorCode() 取得业务码与 HTTP 状态
从 getMessage() 取得本次实际消息
    ↓
ApiResponse.failure(...) 包装 JSON 响应体
ResponseEntity 设置 HTTP 状态并携带响应体
```

捕获和 HTTP 响应转换已经在 `GlobalExceptionHandler` 中实现。业务异常处理器保留 `e.getMessage()`，因此既能返回 ErrorCode 默认消息，也能返回本次自定义消息；未知异常处理则使用固定的安全消息并记录完整堆栈。

## 6. 学习者刚理解的知识

接续讲解时可引用这些结论，但不要直接跳过尚未讲过的 Spring 异常处理概念：

- `new` 创建异常对象，`throw` 抛出已有对象；单独 throw 不会自动再创建一次。
- `this` 指向正在初始化或调用方法的 BusinessException 对象，不是调用它的 Service 对象。
- `this.errorCode = errorCode` 左边是成员字段，右边是构造参数；赋值保存对象引用，不是复制全部字段。
- `final` 限制字段再次赋值，不代表其引用对象一定深度不可变。
- `super(...)` 调用父类构造方法，初始化同一对象的父类状态；不会另外创建一个“父类对象”。
- 异常的消息最终由 Throwable 中的状态保存，通过继承的 `getMessage()` 读取。
- `exception` 或 `ex` 是示例的变量名，没有固定命名要求。
- ErrorCode 的消息是预设的默认消息，异常的消息可以是本次自定义消息。
- `ApiResponse<Void>` 仍返回完整包装对象；Void 作为泛型参数不等于方法返回 void。
- 已讨论静态泛型方法的 `<T>`、类上的 `<T>`、目标类型推断、钻石语法，以及无参 success 的两种方案；正式代码选用返回 `ApiResponse<Void>` 的方案。
- 主键、外键、非空与唯一约束已经通过三张表练习；外键本身不自动要求子表值唯一或非空。

既有 M1 笔记已经补充 BusinessException、全局异常处理、Jackson、日志、泛型、注解和手工验证。接手时仍应结合实际代码核对，不能只凭旧章节标题判断实现状态。

## 7. 本地 Java、Maven 与 jenv

交接时在加载 `.zshrc` 的交互式 zsh 中重新检查：jenv 选择 21，Java 与 javac 均为 21.0.7，Maven 3.9.14 也使用 Java 21.0.7。

学习者现有 `.zshrc` 使用 jenv 初始化并将 `$HOME/paths/maven/bin` 加入 PATH。已明确选择继续由 jenv 管理 Java，**不必为了本项目强制手动设置 JAVA_HOME，也不把启用 jenv export 插件作为阻塞条件**。

助手曾在不加载交互式配置的工具 shell 中看到 Java 17，不能把这种结果直接认定为学习者没切换成功。需要时在真实交互式终端检查，或明确加载交互式 zsh：

```bash
# 使用现有后端目录，避开工具可能保留的旧工作目录
cd /Users/wujiadai/Documents/ChatGPT/HireFlow/backend

# 在加载 .zshrc 的 zsh 中核对 Maven 及其使用的 Java
/bin/zsh -ic 'mvn -version'

# 同样使用该环境编译后端；这条命令不执行自动化测试
/bin/zsh -ic 'mvn compile'
```

不要把 Maven 的 3.9.14 与 Java 的 21.0.7 当作需要相同的版本号。项目 POM 的 Java 21 配置不会自动替换运行 Maven 的 JDK。

宪章中语言栏仍含 Temurin 字样，而本机输出是 Homebrew OpenJDK。本文如实记录实际环境，未改变 Java 21 基线，也未自行修改发行版决策。

## 8. Git 状态与待收尾内容

提交和推送后，应在三个仓库分别执行 `git status -sb`，并用 `git ls-remote origin refs/heads/main` 核对远端 `main`。不要根据聊天中的“已推送”描述代替仓库检查。

本轮计划提交：

| 仓库 | 提交主题 | 主要内容 |
| --- | --- | --- |
| backend | `feat: add unified exception handling` | BusinessException、GlobalExceptionHandler、验证 Controller、DTO、统一响应接入 |
| docs | `docs: complete M1 exception handling notes` | M1 笔记、HANDOFF、路由决策与导航 |

如果 backend 的分支显示 `AM`，表示新文件已有旧暂存内容、工作区又发生了修改。提交前必须同时查看 `git diff` 和 `git diff --cached`，重新暂存最新版本，不能把旧的空类占位提交上去。保留已有修改和无关文件，不擅自 reset、覆盖或批量暂存。

M1 的统一响应、异常处理和基础手工验证已完成；M1 仍需处理通用 HTTP 错误边界：不存在 URL 的 404 和方法不匹配的 405。当前二者会被 `Exception.class` 兜底为 500，完成后才能做 M1 最终验收和阶段总结。正式推送收尾时沿用 [SSH 检查清单](./security/checklists/ssh-security-checklist.md)，核对仓库状态与远端提交；上次安全检查结果不能当作永久实时保证，也不要输出私钥或凭据。

## 9. 接手后的第一个回合

先完成只读核对，再开始讲解；不要立即替学习者生成完整的异常处理器。

1. 阅读四份 Constitution，并核对三个仓库状态及本地 `BusinessException`、`ApiResponse`、`ErrorCode`、`GlobalExceptionHandler`。
2. 先复述当前已完成的主链路：`BusinessException` 抛出 → `@RestControllerAdvice` 捕获 → `ResponseEntity<ApiResponse<Void>>` 返回 JSON。
3. 学习者已经理解“注解本身做什么”的基础问题，接下来应转向 404/405 边界：先解释 `NoResourceFoundException`、`HttpRequestMethodNotSupportedException` 与 `Exception` 兜底之间的关系，再决定错误码和响应结构。
4. 按带练协议说明本步目标、验收标准、宪章依据和依赖；Spring MVC 已由 Spring Boot 3.4.5 提供，优先不新增依赖。
5. 让学习者先写出候选错误码与两个处理器的思路，再审查；不要直接把完整代码贴完。
6. 实现后启动应用，手工检查 404、405、业务异常默认/自定义消息、字段校验、非法 JSON、未知异常和健康接口。仅编译成功不能代替 HTTP 验收。

保持 Controller → Service → Mapper 分层一致；不要提前代写 M2 登录鉴权。

## 10. 复制到新会话的提示词

```text
继续带我做 HireFlow Java 全栈学习项目。

请先读取：
/Users/wujiadai/Documents/ChatGPT/HireFlow/docs/HANDOFF.md
并按该文档的阅读顺序读取 docs/Constitution/ 的四份宪章。

实际项目在 /Users/wujiadai/Documents/ChatGPT/HireFlow/，旧 codeForTest 目录已删除。
请先只读核对三仓库状态及当前代码，保留已有未提交修改和暂存状态。
当前在 M1：统一响应、GlobalExceptionHandler 和基础手工验证已完成，接下来讲解并引导我处理通用 HTTP 404/405 边界。请先解释异常链路和错误码设计，再让我仿写，不要直接给完整模块。
我在 IDEA 中亲手写主要业务代码；请先解释目标、设计原因和依赖，再分小步提示。
示例关键语句前加注释，同类任务让我仿写，不要直接代写完整模块。
当前阶段暂缓自动化测试，使用编译、启动和手工接口验证。
如果你无法访问这些本地文件，请明确说明缺少的内容，不要假装已经读取。
```

本次迁移缘由：旧会话曾在选择 DeepSeek 后先执行自动压缩，日志记录 ModelDownshift、PreTurn，压缩调用仍使用 gpt-6.1-sol，随后收到 404。本文件用于在新会话中从文件接续，不要求新模型继承整段旧会话，也不承诺新供应商的接口和工具调用已完成兼容性验证。
