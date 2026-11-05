# HireFlow 会话交接

> 交接复核：2026-11-05，Asia/Shanghai。
> 代码与 HTTP 验证日期：2026-11-03。
> 用途：更换模型或供应商后，让新会话从本地文件继续带练。
> 当前阶段：M1 已完成；下一步进入 M2 登录鉴权纵向切片。
> 核实原则：本文记录交接时的事实，不替代 Constitution。接手后先只读检查文件和 Git 状态，不要假设之后没有变化。

## 1. 项目入口与阅读顺序

项目根目录是：

```text
/Users/wujiadai/Documents/ChatGPT/HireFlow/
```

外层不是 Git 仓库：

```text
/Users/wujiadai/Documents/ChatGPT/HireFlow/
├── README.md          本地导航，不在 Git 版本管理中
├── TASKS.md           本地任务清单，不在 Git 版本管理中
├── backend/           独立 Git 仓库，GitHub 名称 hireflow-backend
├── frontend/          独立 Git 仓库，GitHub 名称 hireflow-frontend
└── docs/              独立 Git 仓库，GitHub 名称 hireflow-docs
    ├── HANDOFF.md     本交接文档
    └── Constitution/ 项目共识
```

旧目录 `/Users/wujiadai/Documents/ChatGPT/codeForTest/` 已删除。工具如果仍保留这个旧工作目录，执行命令时必须显式指定上面的现有项目目录，不能因此重新初始化项目。

先读四份宪章，再核对当前代码：

1. [Mission.md](./Constitution/Mission.md)：项目目的、成功标准与带练协议。
2. [TechStack.md](./Constitution/TechStack.md)：技术基线、分层和依赖约束。
3. [RoadMap.md](./Constitution/RoadMap.md)：里程碑和验收条件。
4. [DECISIONS.md](./Constitution/DECISIONS.md)：追加式关键决策记录。

随后读取 [M1 学习笔记](./notes/M1-database-and-response-basics.md)；需要设计新表或新模块时，再读 [数据库设计手册](./guides/database-design-playbook.md)。根目录的 `TASKS.md` 和 `README.md` 是本地辅助文件，不在三个子仓库的版本管理中。

## 2. 学习目标与带练规则

学习者即将毕业，目标是准备杭州中小公司的 Java 全栈岗位。他需要在有限时间内亲手完成一条可运行、可演示、能应对面试追问的业务闭环。

当前最需要的不是堆功能，而是补上从“看懂教程”到“能独立设计、实现、排错和解释”的断层。用户明确表示自己不会写也不了解注解、泛型、DTO、异常链路等内容，因此讲解必须具体。

必须遵守：

- 全程使用中文，分小步推进。
- 学习者在本地 IntelliJ IDEA 中亲手编写主要业务代码。
- 每项工作开始前先说明：目标、验收标准、宪章依据、是否需要新增依赖。
- 需要依赖时说明名称、作用、版本来源和引入时机；未经确认不要擅自新增。
- 先讲“业务要求如何推导到设计”，不能只给最终代码。
- 新概念首次出现时给完整示例；关键语句前加简短注释。
- 示例之后解释为什么这样设计、边界和常见错误。
- 同类任务已经示范后，改为检查清单和提示，由学习者先仿写，助手审查；不要直接代写整个业务模块，也不要擅自修改学习者代码。
- 固化为：示例讲解 → 学习者仿写 → 审查 → 编译/启动/手工验证 → Git 提交 → 面试复盘。
- 用户会持续追问“这个东西是哪来的”“为什么能这样写”“类型是什么”。应回到类、接口、方法签名和调用链解释，不能只说“框架会自动处理”。
- 当前阶段暂缓自动化测试；可运行验证使用编译、启动和手工 HTTP 请求。不能把“没有运行测试”写成“测试通过”。
- 阶段完成必须沉淀学习笔记和面试问答。
- 关键技术或范围变更先说明影响并取得确认，再更新 Constitution 与决策日志。
- 普通实现步骤不需要反复询问是否允许继续。

## 3. 已确认的技术与业务边界

| 项目 | 当前基线或文件中的版本 | 说明 |
| --- | --- | --- |
| Java | 21；本机为 Homebrew OpenJDK 21.0.7 | 通过 jenv 管理多版本 |
| Maven | 3.9.x；本机为 3.9.14 | Maven 自身版本与运行它的 Java 版本分开看 |
| Spring Boot | `pom.xml` 为 3.4.5 | 不为迁移会话升级依赖 |
| MyBatis-Plus | `pom.xml` 为 3.5.12 | 依赖已引入，业务 Entity/Mapper 尚未实现 |
| MySQL | 8.4 | 开发库 `hireflow`、预留测试库 `hireflow_test` |
| Lombok | Spring Boot 父 POM 管理，当前为 1.18.38 | 不单独写版本号 |
| JWT | JJWT 0.12.6 | 依赖已引入，登录鉴权尚未实现 |
| 前端 | Vue 3、TypeScript、Vite、Element Plus、Pinia、Vue Router、Axios | 具体版本见 `package.json` 和 `package-lock.json` |

后端遵循 Controller → Service → Mapper；事务和业务规则放在 Service，Controller 不直接调用 Mapper。Entity、DTO、VO 分离。接口文档使用 Markdown，不引入 Swagger。

鉴权规划为 JWT + Spring MVC Interceptor + BCrypt，不使用 Spring Security 过滤器链。BCrypt 尚未引入依赖；M2 需要时先评估，不能直接改成完整 Spring Security。

统一响应结构：

```json
{
  "code": 0,
  "message": "success",
  "data": null
}
```

业务接口统一使用 `/api/v1` 前缀。这个前缀是项目统一约定，不是 Spring 的强制规则；“是谁规定的”这一点的准确答案是：**团队/项目可以规定，行业通常把它作为 REST API 与版本控制的常用惯例**。HireFlow 已在 `DECISIONS.md` 正式采用。

## 4. 实际完成情况

| 范围 | 状态 | 事实 |
| --- | --- | --- |
| M0 环境与三仓库基线 | DONE | 三仓库建立并能独立打开；M0 当轮已有编译、测试、启动和前端构建记录 |
| M1 数据库与基础工程 | DONE | 三张核心表、统一响应、全局异常处理、404/405 边界及手工回归已完成 |
| M1 自动化测试 | 暂缓 | 没有新增或运行自动化测试；不能把手工验证写成“测试通过” |
| M2 登录鉴权 | TODO，尚未开始 | 没有注册、登录、JWT 签发/校验、拦截器或角色路由业务代码 |
| M3-M7 | TODO | 不得提前宣称完成 |

M1 当前已有：

- `users`、`candidate_profiles`、`companies` 三张表及手写 SQL。
- `ApiResponse<T>` 统一成功/失败响应。
- `ErrorCode` 将业务码、默认消息和 HTTP 状态绑定。
- `BusinessException` 与 `GlobalExceptionHandler`。
- 字段校验、非法 JSON、业务异常、未知异常、404、405 的统一处理。
- 405 保留 `Allow` 响应头。
- 健康接口和异常验证接口。
- M1 学习笔记，覆盖注解、泛型、DTO、Jackson、日志和响应分层。

## 5. 当前统一响应与异常设计

核心代码位置：

```text
backend/src/main/java/com/hireflow/common/response/ApiResponse.java
backend/src/main/java/com/hireflow/common/exception/ErrorCode.java
backend/src/main/java/com/hireflow/common/exception/BusinessException.java
backend/src/main/java/com/hireflow/common/exception/GlobalExceptionHandler.java
backend/src/main/java/com/hireflow/health/HealthController.java
backend/src/main/java/com/hireflow/health/ExceptionVerificationController.java
backend/src/main/java/com/hireflow/health/ValidationDemoRequest.java
```

### 5.1 三个类型的职责

```java
// JSON 响应体的模板；T 由调用者决定 data 的类型
public class ApiResponse<T> {
    private final int code;
    private final String message;
    private final T data;
}
```

```java
// Spring 提供的完整 HTTP 响应：状态码 + 响应头 + 响应体
ResponseEntity<ApiResponse<Void>>
```

```java
// HireFlow 的单一错误策略源：业务码、默认消息、HTTP 状态
ErrorCode.NOT_FOUND
```

不要把它们混成一个东西：

| 类型 | 所在层 | 负责什么 | 不负责什么 |
| --- | --- | --- | --- |
| `ErrorCode` | 业务错误配置 | 业务码 `1004`、默认消息、HTTP `404` | 不直接发送响应 |
| `ApiResponse<T>` | JSON 契约 | `{code,message,data}` 的固定结构 | 不知道 HTTP 头，也不决定状态 |
| `ResponseEntity<T>` | HTTP 协议 | 状态行、响应头、响应体 | 不理解业务码含义 |

### 5.2 为什么返回 `ResponseEntity`

控制器方法声明：

```java
public ResponseEntity<ApiResponse<Void>> handleBusinessException(
        BusinessException e) {
```

它表示：这个方法最终返回“一个完整 HTTP 响应，其中 body 的类型是 `ApiResponse<Void>`”。

```java
// status(...) 先选 HTTP 状态，body(...) 再放入 JSON 响应体
return ResponseEntity
        .status(errorCode.getHttpStatus())
        .body(body);
```

`.body(body)` 的返回值正是 `ResponseEntity<ApiResponse<Void>>`，与声明一致。`ResponseEntity.status(...)` 返回的中间对象不是 `void`；它用于链式配置，最后交给 `.body(...)` 完成构造。

### 5.3 业务码与 HTTP 状态码为什么分层

```text
ErrorCode.METHOD_NOT_ALLOWED
├─ 业务码 1006
├─ 默认消息“请求方法不支持”
└─ Spring HttpStatus.METHOD_NOT_ALLOWED（HTTP 405）
        ↓
ApiResponse<Void>（JSON body）
        ↓
ResponseEntity<ApiResponse<Void>>
├─ HTTP 405
├─ Allow: GET
└─ JSON body
```

业务码让前端和日志精确定位业务原因；HTTP 状态让浏览器、网关、监控和通用 HTTP 客户端理解协议语义。只留一个数字会丢掉另一层信息。

`ErrorCode.METHOD_NOT_ALLOWED` 和 `HttpStatus.METHOD_NOT_ALLOWED` 名称相同，但类型完全不同。前者是项目的错误策略对象，后者是 Spring 的标准 HTTP 状态枚举。

## 6. 当前已验证的 HTTP 行为

验证日期：2026-11-03。
环境：Java 21、Spring Boot 3.4.5。
方式：`mvn -q compile` 通过，启动应用后用 `curl` 手工请求；**没有运行自动化测试**。验证后已停止服务。

| 场景 | 结果 |
| --- | --- |
| `GET /api/v1/health` | 200，`code=0` |
| 业务异常，使用默认消息 | 404，`code=1004` |
| 业务异常，自定义消息 | 404，`code=1004`，“职位不存在” |
| 空字段校验 | 400，`code=1001`，字段消息 |
| 非法 JSON | 400，`code=1001`，“请求体格式错误” |
| 未知异常 | 500，`code=5000`，服务端记录完整堆栈 |
| 合法 DTO | 200，空响应体 |
| 不存在的 URL | 404，`code=1004` |
| `POST /api/v1/health` | 405，`code=1006` |
| 同上响应头 | `Allow: GET` |

注意：M1 当轮记录的是手工验证，不是自动化测试通过；接口类型、响应结构和状态码必须以后续实际代码和请求为准。

## 7. 本地 Java/Maven 环境

项目要求 Java 21。`pom.xml` 的 `<java.version>21</java.version>` 只约束编译目标，不会自动替换运行 Maven 的 JDK。

在真实交互式终端中可以这样核对：

```bash
cd /Users/wujiadai/Documents/ChatGPT/HireFlow/backend

# 同时查看 Maven 自身版本和它实际使用的 Java
mvn -version

# 编译后端；这条命令不执行自动化测试
mvn -q compile
```

本机曾出现工具 shell 与交互式 shell 的 Java 版本不一致。不要因为某一次非交互命令显示 Java 17 就断定 IDEA 或学习者终端配置失败。需要时在 IDEA 或加载 `.zshrc` 的真实 zsh 中核对。

## 8. Git 状态与收尾

三个仓库各自独立，必须分别检查 `git status -sb`，并核对远端 `main`：

```bash
cd /Users/wujiadai/Documents/ChatGPT/HireFlow/backend
git status -sb
git rev-parse HEAD
git ls-remote origin refs/heads/main

cd ../frontend
git status -sb
git rev-parse HEAD
git ls-remote origin refs/heads/main

cd ../docs
git status -sb
git rev-parse HEAD
git ls-remote origin refs/heads/main
```

本次 M1 收尾的提交事实：

| 仓库 | 内容 | 提交 |
| --- | --- | --- |
| backend | 404/405 专用异常处理 | `f0da42e feat: handle HTTP 404 and 405 errors` |
| docs | M1 宪章、笔记、交接与 README 收尾 | 以本次 docs 最终提交为准 |
| frontend | M1 无代码变更 | `e1d9820 chore: bootstrap Vue frontend shell` |

根目录 `README.md` 和 `TASKS.md` 不在 Git 仓库中，不能通过 GitHub 提交；需要本机保存。

安全收尾沿用 [SSH 检查清单](./security/checklists/ssh-security-checklist.md)。不要输出私钥、数据库密码或 JWT 密钥。

## 9. 接手后的第一个回合：开始 M2

接手后先只读核对：四份宪章、M1 笔记、三个仓库状态、现有后端代码。不要直接开始写完整登录模块。

第一个回合的目标不是交付所有 M2 功能，而是完成 **M2 注册纵向切片的最小设计**，并让学习者先仿写。按以下顺序带练：

1. 说明 M2 的范围和本轮验收标准：
   - 注册接口；
   - 输入校验；
   - 邮箱归一化与重复检查；
   - 密码哈希；
   - 用户落库；
   - 统一成功/失败响应。
2. 说明依据：
   - `TechStack.md`：Controller → Service → Mapper、Entity/DTO/VO 分离、HTTP + 业务码分层；
   - `RoadMap.md`：M2 包含注册、登录、JWT、角色路由、个人资料、前后端联调；
   - `DECISIONS.md`：`/api/v1` 前缀、异常与响应约定；
   - 当前阶段暂缓自动化测试，使用编译、启动和手工 HTTP 验证。
3. 先核对依赖：
   - 现有 JJWT 依赖已存在；
   - BCrypt 目前尚未引入；不要擅自新增 Spring Security 全链路，先评估是否需要 `spring-security-crypto`；
   - 如果新增依赖，先说明名称、作用、版本来源和引入时机并取得确认。
4. 先分析注册切片：
   - API 建议：`POST /api/v1/auth/register`；
   - 请求 DTO 至少考虑邮箱、密码、姓名、角色；
   - 角色只能为 `CANDIDATE` 或 `RECRUITER`；
   - 邮箱要先归一化，再检查唯一性；
   - 密码只保存 BCrypt 哈希，绝不保存明文；
   - Service 负责业务规则和事务，Controller 不直接操作 Mapper；
   - 响应使用 `ApiResponse`，错误使用 `ErrorCode`/`BusinessException`。
5. 在学习者写代码前，先讨论并确认几个边界：
   - 注册 `CANDIDATE` 时，是否同时创建 `candidate_profiles` 空资料；
   - 邮箱大小写和首尾空格如何处理；
   - 重复邮箱返回哪个业务码；
   - 密码长度和复杂度规则；
   - 注册成功返回用户信息还是直接返回 token；
   - 失败时如何避免暴露“邮箱已存在”以外的内部信息。
6. 让学习者先写 DTO、Service 接口/实现和 Controller 的骨架；助手逐段审查，不能把整个模块一次性代写完。
7. 实现后先用编译、启动和 `curl` 验证注册成功、重复邮箱、字段校验、非法 JSON 和错误响应，再决定是否进入登录与 JWT 切片。

当前 M2 尚未开始编写任何业务代码。不要把规划写成已完成。

## 10. 复制到新会话的提示词

```text
继续带我做 HireFlow Java 全栈学习项目。

请先读取：
/Users/wujiadai/Documents/ChatGPT/HireFlow/docs/HANDOFF.md
并按该文档的阅读顺序读取 docs/Constitution/ 的四份宪章。

实际项目在 /Users/wujiadai/Documents/ChatGPT/HireFlow/，旧 codeForTest 目录已删除。
请先只读核对三仓库状态及当前代码，保留已有修改和暂存状态。
当前 M1 已完成；请从 M2 登录鉴权的“注册纵向切片”开始带练，不要继续处理 404/405，也不要直接代写完整登录模块。

先向我说明本轮目标、验收标准、宪章依据、所需依赖和设计思路，再让我在 IDEA 中仿写。
新概念和关键语句要给示例并加注释；我追问类型、注解、泛型、DTO、异常链路时，请回到类和方法签名解释。
当前阶段暂缓自动化测试，使用编译、启动和手工接口验证。
如果你无法访问这些本地文件，请明确说明缺少的内容，不要假装已经读取。
```

本次交接修订原因：旧版 HANDOFF 仍把 M1 的 404/405 写成待处理，会把新会话引回已完成工作。本版已按当前仓库事实改为 M1 DONE、M2 TODO。
