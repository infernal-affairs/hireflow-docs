# M1：数据库与统一响应基础

- 日期：2026-10-29，补充更新：2026-11-03，交接复核：2026-11-05
- 状态：M1 已完成；统一响应、异常处理、404/405 路由边界和手工验收均已收尾
- 依据：`Constitution/Mission.md`、`TechStack.md`、`RoadMap.md`

## 一、本阶段完成内容

> 阅读提示：基础数据库与统一响应从第二节开始；若重点关注本轮异常处理、注解和追问，可直接跳到第十二节，再回看第五节“今天遇到的错误和修正”。

- 在 MySQL 中创建 `hireflow` 和 `hireflow_test` 数据库。
- 在 `hireflow` 中创建：
  - `users`
  - `candidate_profiles`
  - `companies`
- 设计并实现后端统一响应类 `ApiResponse<T>`。
- 使用 Java 21 编译通过。
- 实现统一异常处理，并用启动后的 HTTP 请求完成主场景验证。
- 补齐路由级错误：不存在的 URL 返回 404，HTTP 方法不匹配返回 405，并保留 405 的 `Allow` 响应头。
- 当前阶段暂缓自动化测试，以编译、启动和手工接口验证作为功能推进门槛。

## 二、数据库表设计

### `users`

核心字段：

- `id`：自增主键。
- `email`：登录邮箱，非空且唯一。
- `password_hash`：密码哈希，不保存明文。
- `name`：用户姓名。
- `role`：`CANDIDATE` 或 `RECRUITER`。
- `created_at`、`updated_at`：创建和更新时间。

关键约束：

- `uk_users_email` 保证一个邮箱只能注册一个账号。
- `chk_users_role` 限制角色取值。
- `updated_at` 使用 `ON UPDATE CURRENT_TIMESTAMP(3)` 自动维护。

### `candidate_profiles`

核心字段：

- `user_id`：用户 ID，同时作为主键和外键。
- `skills`：逗号分隔的技能，可空。
- `bio`：个人简介，可空。
- `created_at`、`updated_at`：创建和更新时间。

关键设计：

- 业务规则是“一人一份求职者资料”，所以 `user_id` 直接做主键。
- 外键引用 `users.id`，并使用 `ON DELETE RESTRICT`、`ON UPDATE RESTRICT`。
- 数据库只能保证资料属于真实用户，不能保证该用户一定是 `CANDIDATE`；角色校验必须放在 Service 层。

### `companies`

核心字段：

- `id`：自增主键。
- `display_name`：展示名称。
- `normalized_name`：归一化名称，唯一。
- `created_by`：创建人用户 ID，外键引用 `users.id`。
- `created_at`、`updated_at`：创建和更新时间。

关键设计：

- `display_name` 用于展示，不加唯一约束。
- `normalized_name` 用于判重，加唯一约束。
- 创建人与公司成员关系后续由 `company_members` 表达，不能混为一张表。

## 三、统一响应 `ApiResponse<T>`

当前结构：

```text
{
  code,
  message,
  data
}
```

当前实现要点：

- 使用 Lombok `@Getter` 和 `@AllArgsConstructor` 生成 getter 和全参构造方法。
- 字段使用 `private final`，创建后不可修改。
- 通过 Lombok 生成 getter，供 Jackson 序列化和调用方读取。
- 成功业务码统一为 `0`。
- 默认成功消息为 `"success"`。
- 提供带数据和无数据两个成功工厂方法。
- 使用 `@JsonInclude(JsonInclude.Include.ALWAYS)`，保证 `data = null` 时仍输出该字段。

## 四、今天重点理解的知识

### 1. 主键和外键

- 主键唯一标识当前表的一行，要求非空、唯一。
- 外键表示子表字段的值必须存在于父表被引用字段中。
- 外键字段本身不要求唯一，也不一定非空。
- 父表被引用字段应能唯一确定一行，通常选择主键。

### 2. 一对一关系的表达

`candidate_profiles.user_id` 同时作为主键和外键，表达：

- 一个用户最多一份资料。
- 资料必须属于真实用户。

如果新增独立 `profile_id`，同时允许多个相同 `user_id`，就变成了一对多。

### 3. 字段约束如何确定

判断顺序：

- 不能为空：`NOT NULL`
- 不能重复：`UNIQUE`
- 只能取固定值：`CHECK`
- 引用其他表：`FOREIGN KEY`
- 经常查询：索引
- 需要默认值：`DEFAULT`

数据库约束负责最终兜底，Service 层负责友好提示和业务规则。

### 4. 展示名称和归一化名称

- `display_name` 是展示值，应保留用户可读写法。
- `normalized_name` 是比较值，应统一大小写和空格后再判重。
- 唯一约束应加在 `normalized_name` 上。
- 展示名称和比较值分开，避免展示需求影响判重规则。

### 5. 表选项

- `ENGINE` 是表级属性，不由 `CREATE DATABASE` 决定。
- MySQL 8 默认存储引擎通常是 InnoDB，但依赖默认配置不够显式。
- `utf8mb4` 支持完整 Unicode。
- `utf8mb4_0900_ai_ci` 是比较规则，`ai` 忽略重音，`ci` 忽略大小写。
- 数据库已设置字符集时，表可以继承，但显式声明更可复现。
- `COMMENT` 只用于结构说明，不参与业务逻辑。

### 6. 表单字段和响应字段的边界

- Entity 对应数据库表，不应直接返回给前端。
- DTO 接收请求参数，承载校验注解。
- VO 返回前端，只暴露需要的字段。
- 统一响应只负责包装，不负责业务转换。

### 7. `NULL` 和空字符串

- `NULL` 表示没有值，适合“尚未填写”。
- 空字符串表示有值但内容为空。
- 选填字段是否使用 `NULL` 要根据业务语义决定。
- 查询 `NULL` 必须使用 `IS NULL`，不能使用 `= NULL`。

### 8. Java 泛型

- 类上的 `T` 是类级类型参数；每个 `ApiResponse<String>`、`ApiResponse<UserVO>` 在使用处确定自己的 `T`。
- 泛型让编译器检查“期望类型”和“实际放入的数据类型”是否一致。
- 静态方法不能直接使用类上的 `T`，需要自己声明方法级 `<T>`。
- `new ApiResponse<>(0, "success", data)` 的钻石操作符会由编译器根据上下文推断类型。
- Java 泛型主要提供编译期类型安全，运行时会发生类型擦除。
- 无参数成功方法固定返回 `ApiResponse<Void>`，避免依赖上下文推断。

可把它想成“带类型标签的盒子”：`ApiResponse<String>` 的 `data` 只接受 `String`，`ApiResponse<Integer>` 的 `data` 只接受 `Integer`。盒子结构相同，但类型标签不同，编译器会分别检查。

### 9. `void` 和 `Void`

- `void` 是方法不返回值的语法。
- `Void` 是类，可以作为泛型参数。
- `ApiResponse<Void>` 返回的仍是完整响应对象，只是 `data` 为 `null`。

### 10. 静态工厂方法

静态工厂方法把响应构造规则集中到一个位置：

- 统一成功码 `0`。
- 统一默认消息 `"success"`。
- 减少 Controller 重复写参数和出错概率。
- 保持构造方法可复用，但业务代码优先使用工厂方法。

### 11. 三层架构

- Controller 处理 HTTP、参数和响应。
- Service 处理业务规则、权限、事务和 Entity 到 VO 的转换。
- Mapper 只负责数据库访问。
- Controller 不直接调用 Mapper。
- 事务加在 Service 层。
- 后续使用标准分层包结构。

## 五、今天遇到的错误和修正

### 1. `final` 字段未初始化

现象：给 `ApiResponse` 字段加 `final` 后编译报错。

原因：`final` 字段必须在对象创建完成前赋值。

解决：增加全参构造方法，在构造方法中初始化三个字段。

### 2. `success()` 误写成泛型方法

初稿：

```java
public static <T> ApiResponse<T> success()
```

项目采用方案 A，修正为：

```java
public static ApiResponse<Void> success()
```

原因：无数据时不需要根据调用上下文推断 `T`。

### 3. `AUTO_INCREMENT` 放错字段

初稿将 `AUTO_INCREMENT` 写在 `display_name` 上。

原因：`AUTO_INCREMENT` 只能用于整数列，且该列必须是索引或主键的一部分。

修正：放到公司主键 `id` 上。

### 4. 建表脚本重复执行报错

原因：`CREATE TABLE` 在表已存在时会失败。

开发阶段且无数据时，使用：

```sql
DROP TABLE IF EXISTS 表名;
```

再重新执行建表脚本。有数据时应使用 `ALTER TABLE`，不能直接删表。

### 5. 聊天中的 `*CURRENT_TIMESTAMP*`

聊天渲染可能显示星号，但实际项目文件中是：

```sql
CURRENT_TIMESTAMP(3)
```

先检查文件真实内容，不要根据聊天渲染直接判断语法错误。

### 6. IDEA 类名黄色波浪线

`mvn compile` 已通过，说明不是编译错误。

可能是 IDEA 建议将不可变数据类改为 record，或 Git 未提交提示。当前保留普通 class，不自动转换。

## 六、面试问答

1. 为什么邮箱既要 `NOT NULL` 又要唯一索引？
2. 为什么 `candidate_profiles.user_id` 可以同时做主键和外键？
3. 外键字段一定唯一、一定非空吗？
4. 为什么 `display_name` 不唯一，而 `normalized_name` 唯一？
5. 为什么数据库约束不能替代 Service 校验？
6. 为什么 Controller 不直接调用 Mapper？
7. 为什么事务加在 Service 层？
8. `ApiResponse<T>` 的泛型有什么作用？
9. `void` 和 `Void` 有什么区别？
10. 为什么业务成功码使用 `0`，HTTP 成功状态码使用 `200`？
11. 静态工厂方法相比直接 `new` 有什么好处？
12. 为什么 `data = null` 时仍要保留 `data` 字段？

## 七、已完成与后续事项

- 通用 HTTP 404/405 边界已处理：不存在的 URL 返回 `1004/404`，方法不匹配返回 `1006/405`，405 保留 `Allow`。
- 当前异常处理主链路已完成，学习笔记已将本轮知识集中记录在本文件。
- 当前阶段暂缓自动化测试；后续是否补测需单独决策。
- MySQL 使用本机账号密码，仓库中不记录真实凭据。

## 八、相关提交

- backend：`ae40d39` 加入 M1 核心表与 `ApiResponse`；`88a3920` 加入统一异常处理；`f0da42e` 补齐 404/405 路由边界；`4a3f595` 同步 M1 完成状态。
- docs：`9038bec` 记录 M1 共识与学习笔记；`a67468c` 完成异常处理笔记；`9b1d431` 完成 M1 收尾与 M2 交接。
- frontend：M1 无代码变更。

## 九、代码实现补充记录

### `ApiResponse` 已实现的约定

- `code`：`int`，成功为 `0`。
- `message`：`String`，成功默认为 `"success"`。
- `data`：泛型 `T`，没有数据时为 `null`。
- 三个字段均为 `private final`，通过全参构造方法初始化。
- 提供 `getCode()`、`getMessage()`、`getData()`。
- 提供 `success(T data)` 和 `success()` 两个静态工厂方法。
- 类上使用 `@JsonInclude(JsonInclude.Include.ALWAYS)`，防止 `data = null` 被全局 Jackson 配置省略。

### 已补齐

- `failure` 静态工厂方法。
- `ErrorCode` 错误码设计。
- 全局异常处理。
- 第一个返回统一响应的 Controller 手工验证。

统一异常处理的实现细节、实测结果、注解速查和面试问答见文末“统一异常处理与手工验收”章节。

### 本轮提交

- backend：`feat: add unified exception handling`；后续 `feat: handle HTTP 404 and 405 errors` 补齐路由边界；`docs: align backend status with M1 completion` 同步仓库状态。
- docs：`docs: complete M1 exception handling notes`；`docs: finalize M1 and prepare M2 handoff` 完成阶段收尾。

提交哈希以远端仓库为准。

## 十、Lombok 对比结论

- 普通 Java class 的构造方法和 getter 全部可见，但样板代码较多。
- Lombok 版本通过 `@Getter` 和 `@AllArgsConstructor` 生成同样的方法，泛型仍然有效。
- Lombok 不会生成静态工厂方法，`success(...)` 仍需手写。
- `ErrorCode` 使用 `@Getter`，但保留显式构造方法，避免枚举项与字段的绑定关系被隐藏。
- Lombok 版本由 Spring Boot `3.4.5` 管理，当前为 `1.18.38`。
- IDEA 需要启用 Annotation Processing；Maven 编译成功可作为额外验证。

## 十一、ErrorCode 已实现内容

- `INVALID_PARAM(1001, "参数校验失败", HttpStatus.BAD_REQUEST)`
- `UNAUTHORIZED(1002, "未登录或过期", HttpStatus.UNAUTHORIZED)`
- `FORBIDDEN(1003, "无权限执行此操作", HttpStatus.FORBIDDEN)`
- `NOT_FOUND(1004, "请求的资源不存在", HttpStatus.NOT_FOUND)`
- `CONFLICT(1005, "数据冲突", HttpStatus.CONFLICT)`
- `METHOD_NOT_ALLOWED(1006, "请求方法不支持", HttpStatus.METHOD_NOT_ALLOWED)`
- `INTERNAL_ERROR(5000, "服务器内部错误", HttpStatus.INTERNAL_SERVER_ERROR)`

错误码枚举将业务码、默认消息和 HTTP 状态码绑定在一起，后续由全局异常处理器统一使用。`ErrorCode` 的字段带有 Lombok `@Getter`；构造方法仍手写，因此不需要 `@AllArgsConstructor`。

注意两个 `METHOD_NOT_ALLOWED` 的类型不同：

- `ErrorCode.METHOD_NOT_ALLOWED` 是 HireFlow 的错误策略对象，提供业务码 `1006`、默认消息和 HTTP 状态。
- `HttpStatus.METHOD_NOT_ALLOWED` 是 Spring 的 HTTP 标准状态枚举，值对应 405。
- 前者被放进 JSON，后者决定 HTTP 响应行；名称相同不代表它们是同一个值，也不能互相替换。

## 十二、统一异常处理与手工验收（2026-11-02 至 2026-11-03）

### 1. 本步目标与依赖边界

目标：把业务失败、参数校验失败、请求 JSON 无法解析和未知服务异常，都转换为统一的 `{ code, message, data }` 响应，同时保证 HTTP 状态码正确。

依据：

- `Mission.md`：以可运行、可验证和能解释设计作为实践标准。
- `TechStack.md`：统一响应结构、标准 HTTP 状态码、Controller → Service → Mapper 分层。
- `RoadMap.md`：M1 的验收产出包含统一响应和异常处理。
- `DECISIONS.md`：2026-11-01 确认业务接口和临时验证接口统一使用 `/api/v1` 前缀。

依赖：没有新增依赖。`spring-boot-starter-web` 已提供 Spring MVC、Jackson 和 SLF4J；`spring-boot-starter-validation` 已提供 Jakarta Bean Validation。

### 2. 一次请求的完整异常链路

```text
客户端发送 HTTP 请求
    ↓
DispatcherServlet 找到 Controller 方法
    ↓
Spring MVC 解析参数、执行 Bean Validation、调用 Controller
    ↓
Controller 或 Service 抛出异常
    ↓
ExceptionHandlerExceptionResolver 查找 @ExceptionHandler
    ↓
GlobalExceptionHandler 构造 ApiResponse<Void>
    ↓
ResponseEntity 设置 HTTP 状态和响应体
    ↓
Jackson 把 ApiResponse 序列化为 JSON
```

这里有两条彼此独立的结果：

- HTTP 状态：例如 `400`、`404`、`500`，属于 HTTP 协议层。
- JSON 业务码：例如 `1001`、`1004`、`5000`，属于 HireFlow 自己的业务约定。

因此业务异常可以返回 HTTP `404`，同时 JSON 中写业务码 `1004`。它们不是同一套编号。

### 3. `BusinessException` 为什么继承 `RuntimeException`

```java
public class BusinessException extends RuntimeException {
```

`RuntimeException` 是“非受检异常”：

- Service 方法抛出 `BusinessException` 时，不需要在方法签名写 `throws`。
- 调用链中不必逐层捕获，可以交给全局异常处理器统一处理。
- 它是“业务规则明确预期的失败”，例如“职位不存在”，不是程序故障。

双参数构造方法让同一个业务类型可以使用本次上下文相关的消息：

```java
public BusinessException(ErrorCode errorCode, String message) {
    super(message);             // 保存本次实际消息
    this.errorCode = errorCode; // 保存业务码和 HTTP 状态
}
```

之后通过继承 Throwable 得到的 `getMessage()` 读取消息，不再额外定义 `message` 字段，避免同一状态存两份。

### 4. `@RestControllerAdvice` 与 `@ExceptionHandler`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
```

`@RestControllerAdvice` 是全局 Controller 增强组件，同时包含 `@ResponseBody` 语义。异常处理方法返回的对象会被 Jackson 写入响应体，而不是被当成页面名。

```java
@ExceptionHandler(BusinessException.class)
public ResponseEntity<ApiResponse<Void>> handleBusinessException(
        BusinessException e) {
```

`@ExceptionHandler` 声明“这个方法能处理哪种异常”。Spring 根据异常类型匹配处理方法，存在多个候选时选择更具体的匹配。

当前匹配关系：

| 异常 | 处理器 | HTTP | 业务码 |
| --- | --- | --- | ---: |
| `BusinessException` | `handleBusinessException` | 由 ErrorCode 决定 | 由 ErrorCode 决定 |
| `MethodArgumentNotValidException` | `handleMethodArgumentNotValidException` | 400 | 1001 |
| `HttpMessageNotReadableException` | `handleHttpMessageNotReadableException` | 400 | 1001 |
| `NoResourceFoundException` | `handleNoResourceFoundException` | 404 | 1004 |
| `HttpRequestMethodNotSupportedException` | `handleHttpRequestMethodNotSupportedException` | 405 | 1006 |
| 其他 `Exception` | `handleException` | 500 | 5000 |

兜底的 `Exception.class` 不应抢走前面这些更具体的异常，因为 Spring 会优先匹配更具体的异常类型。两个路由级异常处理器也必须放在兜底之前，否则它们会被当成普通 500。

`HttpStatus` 的数值不是我们随意编的：`HttpStatus.NOT_FOUND` 是 Spring 对标准 HTTP `404 Not Found` 的封装，`HttpStatus.METHOD_NOT_ALLOWED` 对应标准 `405 Method Not Allowed`。我们用枚举名表达意图，而不是在业务代码里散落数字。

### 5. `ResponseEntity<ApiResponse<Void>>` 到底是什么

```java
public ResponseEntity<ApiResponse<Void>> handleBusinessException(...)
```

外层 `ResponseEntity` 表示完整的 HTTP 响应，主要包含：

- HTTP 状态码
- HTTP 响应头
- 响应体

内层 `ApiResponse<Void>` 才是业务 JSON：

```json
{
  "code": 1004,
  "message": "职位不存在",
  "data": null
}
```

```java
return ResponseEntity
        .status(errorCode.getHttpStatus())
        .body(body);
```

- `ResponseEntity.status(...)` 设置 HTTP 状态。
- `.body(body)` 把 `ApiResponse` 设为响应体。
- 返回的类型正是声明中的 `ResponseEntity<ApiResponse<Void>>`。

常见困惑：`ResponseEntity.status(...)` 返回的不是 `void`，它返回一个可以继续调用 `.body(...)` 的构建对象；最终 `.body(...)` 返回的就是 `ResponseEntity<ApiResponse<Void>>`。

`ResponseEntity` 是 Spring 提供的类，不是项目自己定义的：

```java
import org.springframework.http.ResponseEntity;
```

它表示“一整个 HTTP 响应”，包含状态码、响应头和响应体。常用的读取方法有：

```java
response.getStatusCode(); // HTTP 状态，例如 405
response.getHeaders();    // 响应头，例如 Allow: GET
response.getBody();       // Java 响应体，例如 ApiResponse 对象
```

### 5.1 “业务码和 HTTP 状态码”为什么要嵌套三层

先明确：并不是同一个业务码被嵌套了三次，而是三个对象各自负责一件不同的事。

以 `ErrorCode.METHOD_NOT_ALLOWED` 为例：

```text
ErrorCode.METHOD_NOT_ALLOWED
├─ 业务码：1006
├─ 默认消息：请求方法不支持
└─ HTTP 状态：HttpStatus.METHOD_NOT_ALLOWED（405）
          ↓ 交给处理器
ApiResponse<Void>
└─ JSON 响应体：{"code":1006,"message":"请求方法不支持","data":null}
          ↓ 放进 ResponseEntity
ResponseEntity<ApiResponse<Void>>
├─ HTTP 状态：405
├─ 响应头：Allow: GET
└─ 响应体：上面的 ApiResponse
```

代码中的对应关系：

```java
ErrorCode errorCode = ErrorCode.METHOD_NOT_ALLOWED;   // 1. 选择错误策略
String message = errorCode.getMessage();               // 使用默认消息
ApiResponse<Void> body = ApiResponse.failure(errorCode, message); // 2. 构造 JSON
return ResponseEntity
        .status(errorCode.getHttpStatus())             // 3. 设置 HTTP 状态
        .headers(e.getHeaders())                       //    保留 Allow 等协议头
        .body(body);                                   //    放入 JSON 响应体
```

三层各自的职责：

| 层 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| `ErrorCode` | 业务码、默认消息、HTTP 状态的集中配置 | 不直接生成 HTTP 响应 |
| `ApiResponse` | JSON 响应的固定结构 `{code,message,data}` | 不知道 HTTP 状态，也不知道响应头 |
| `ResponseEntity` | 完整 HTTP 响应的状态、头和体 | 不关心业务码的含义 |

如果不分层，会出现两类常见问题：

1. 每个 Controller 都手写 `status(405)`、业务码和消息，容易改一处漏一处。
2. 只返回 `ApiResponse`，可能让 Spring 默认使用 200，导致客户端把失败当成成功；或者为了设状态而到处直接返回 `ResponseEntity`，重复业务判断。

因此当前设计的思路是：

```text
业务错误怎么表达 → ErrorCode
客户端看到的 JSON 长什么样 → ApiResponse
HTTP 协议层怎么发送 → ResponseEntity
```

这不是“越深越高级”，而是把变化点分开。以后新增错误时，通常只需在 `ErrorCode` 增加一个常量，并增加对应的异常映射。

另外，`ErrorCode` 和 `HttpStatus` 的名称可能相同，例如都有 `METHOD_NOT_ALLOWED`，但类型和作用不同：

```java
ErrorCode.METHOD_NOT_ALLOWED  // HireFlow 错误策略，带 1006 业务码
HttpStatus.METHOD_NOT_ALLOWED // 标准 HTTP 状态，值为 405
```

业务码用于让前端或调用方精确定位业务原因；HTTP 状态用于网关、浏览器、监控和通用 HTTP 客户端判断请求结果。两者不能合并成一个数字，否则会丢掉其中一层语义。

### 6. Bean Validation 与请求 DTO

```java
public class ValidationDemoRequest {
    @NotBlank(message = "名称不能为空")
    private String name;

    // Jackson 创建对象时通常需要一个可访问的无参构造方法
    public ValidationDemoRequest() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

DTO 的作用是承载一次请求的数据，不映射数据库表。这里的 Controller 参数：

```java
public void validation(
        @Valid @RequestBody ValidationDemoRequest request) {
```

- `@RequestBody`：告诉 Spring 从 HTTP 请求体读取 JSON，再由 Jackson 转成 DTO。
- `@Valid`：告诉 Bean Validation 执行 DTO 字段上的约束。
- `@NotBlank`：字符串不能是 `null`、空串或只有空白字符。
- `@ExceptionHandler(MethodArgumentNotValidException.class)`：校验失败后统一转换成 `1001/400` 响应。

当前只取第一个 `FieldError`：

```java
FieldError fieldError = e.getBindingResult().getFieldError();
```

因此返回如：

```json
{"code":1001,"message":"name:名称不能为空","data":null}
```

多个字段同时非法时，目前只展示其中一个。这不是“所有错误都只能取第一个”的框架规则，而是当前实现的选择；以后可以改成收集全部错误，但要先设计响应数据结构和前端展示方式。

### 7. `e.getMessage()`、默认消息和自定义消息

在参数校验处理器里：

```java
String message = errorCode.getMessage();
if (fieldError != null) {
    message = fieldError.getField() + ":" + fieldError.getDefaultMessage();
}
```

- 初始消息是 `ErrorCode.INVALID_PARAM` 的默认值 `"参数校验失败"`。
- 如果找到了字段错误，就用 DTO 注解中的更具体消息覆盖。
- 如果以后校验异常没有字段错误，则保留安全的默认消息。

非法 JSON 处理器曾经错误地写成：

```java
String message = errorCode.getMessage();
```

这会把默认的“参数校验失败”当成 JSON 语法错误的消息。后来改成：

```java
String message = "请求体格式错误";
```

原因是同样的业务码 `1001` 可以对应不同的具体失败原因，消息可以针对场景覆盖，而业务码保持不变。

业务异常则不同：

```java
ApiResponse.failure(errorCode, e.getMessage());
```

它需要保留 `BusinessException` 中本次的自定义消息，例如“职位不存在”，不能永远使用 ErrorCode 的默认消息。

### 8. `HttpMessageNotReadableException` 与 Jackson

Jackson 负责 JSON 与 Java 对象之间的转换：

- 反序列化：JSON → Java 对象，例如请求 DTO。
- 序列化：Java 对象 → JSON，例如 `ApiResponse` 返回给前端。

非法的：

```json
{bad json}
```

无法被 Jackson 解析，Spring 会抛出 `HttpMessageNotReadableException`。全局处理器把它转换为：

```json
{"code":1001,"message":"请求体格式错误","data":null}
```

不要把 Jackson 解析错误直接返回给客户端，因为异常文本可能包含内部类型和字段细节。

Jackson 的逆过程——把对象写成 JSON——也解释了为什么 `ApiResponse` 需要 getter：序列化时需要有可读取的属性方法。`@JsonInclude(JsonInclude.Include.ALWAYS)` 保证即使全局配置倾向于省略 null，`data: null` 仍然保留，满足统一响应结构约定。

### 8.1 非法 JSON 的异常链路

一次非法 JSON 请求可以按下面的顺序理解：

```text
POST /api/v1/exception-demo/validation
    ↓
@RequestBody 让 Jackson 读取请求体
    ↓
Jackson 解析失败，Spring 抛出 HttpMessageNotReadableException
    ↓
@ExceptionHandler(HttpMessageNotReadableException.class) 匹配
    ↓
构造 ApiResponse.failure(INVALID_PARAM, "请求体格式错误")
    ↓
ResponseEntity.status(400).body(body)
```

`e.getMessage()` 在这里可能包含 Jackson 的类名、字段名和解析位置，不适合直接返回给客户端。因此响应使用固定安全消息；当前处理器没有为这个已预期场景单独打印日志，后续若需要排查客户端请求格式问题，可以只记录必要上下文，并避免记录敏感请求体。

### 9. 为什么记录完整异常，以及日志对象怎样声明

```java
// 类加载时创建一个共享的日志对象；Logger 名称使用当前类的全限定名
private static final Logger log =
        LoggerFactory.getLogger(GlobalExceptionHandler.class);
```

- `LoggerFactory.getLogger(...)`：向 SLF4J 获取 Logger。
- 传入 `GlobalExceptionHandler.class`：日志名称自动使用完整类名，重构时不易写错。
- `private`：只允许当前类使用。
- `static`：属于类，而不是每个处理器对象各存一份。
- `final`：初始化后不再更换 Logger 引用。

SLF4J 是日志门面（统一 API），真正把日志输出到哪里由底层实现决定。Spring Boot 默认集成 Logback，所以应用启动后可以直接在控制台看到 INFO、WARN、ERROR 日志；不需要为了当前 `log.error(...)` 额外写一条 `LoggerFactory.getLogger(...)` 调用。

真正记录完整堆栈的是：

```java
// 第二个参数传入异常对象，日志框架会保留类型、消息、调用栈和 Cause
log.error("未处理的异常", e);
```

不是：

```java
// 只把 getMessage() 拼成字符串，类型和调用栈都会丢失
log.error("未处理的异常" + e.getMessage());
```

这里有三条不同通道：

```text
log.error(...)              → 服务端控制台/日志文件，保留内部细节
ApiResponse.failure(...)    → 客户端响应，只给安全消息
HTTP 状态码                 → 协议层，例如 500
```

所以未知异常的响应是安全的 `5000/500`，而服务端仍然能看到：

```text
java.lang.IllegalStateException: 模拟数据库连接失败：敏感内部信息
    at com.hireflow.health.ExceptionVerificationController.unknown(...)
```

### 9.1 实际业务中会有多少 `GlobalExceptionHandler` 对象

典型情况下，整个 Spring 应用只有一个 `GlobalExceptionHandler` Bean。`@RestControllerAdvice` 注册的是 singleton 作用域的组件，不是每个 Controller、每个请求各创建一个副本。

```text
多个并发请求
    ↓
同一个 GlobalExceptionHandler 实例
    ↓
每个请求调用拥有独立的局部变量 e、body、errorCode
```

因此不要在处理器里增加类似 `private Exception currentException` 的请求级共享字段，否则并发请求会互相覆盖。`static final Logger` 例外：它没有请求状态，且 SLF4J Logger 支持并发使用，所以一个类的多个对象、多个线程可以安全共用。

更准确地说，Bean 实例可以共享，但方法里的局部变量不能跨请求共享；需要请求级数据时应通过参数、返回值或 Spring 的 request scope 显式传递。

### 10. 路由级 404/405 边界（2026-11-03 已处理）

本次通过 `@ExceptionHandler(Exception.class)` 统一了未知业务代码抛出的异常。路由级异常也曾经进入兜底并返回 500：

- 访问不存在的 URL 会触发 `NoResourceFoundException`，修复后返回 `NOT_FOUND/1004/404`。
- 请求方法不匹配会触发 `HttpRequestMethodNotSupportedException`，修复后返回 `METHOD_NOT_ALLOWED/1006/405`。

关键区别：

| 场景 | 请求例子 | 说明 | 修复后 |
| --- | --- | --- | --- |
| URL 不存在 | `GET /api/v1/does-not-exist` | 路径没有匹配到资源 | 404 + `1004` |
| HTTP 方法错误 | `POST /api/v1/health` | 路径存在，但该资源只接受 GET | 405 + `1006` + `Allow: GET` |

不能把两者都笼统处理成“请求不存在”：405 表示“这个 URL 我知道，但你用的方法不对”，客户端可以根据 `Allow` 改用正确的方法。

405 的标准语义要求响应带上允许的方法。Spring 的 `HttpRequestMethodNotSupportedException` 本身提供了 `getHeaders()`，其中包含 `Allow`；处理器把它传给 `ResponseEntity.headers(...)`，避免自定义统一响应时丢掉这个协议头：

```java
return ResponseEntity
        .status(errorCode.getHttpStatus())
        .headers(e.getHeaders())
        .body(body);
```

这里的 `e.getHeaders()` 来自 Spring 异常对象，不是 `ErrorCode`。`ErrorCode` 只决定业务码、默认消息和 HTTP 状态；协议头仍由具体异常提供。

### 11. 自定义异常与未知异常的区别

```java
// 业务预期失败：保留可展示给用户的具体原因
throw new BusinessException(ErrorCode.NOT_FOUND, "职位不存在");

// 未预期故障：只记录细节，客户端只收到安全消息
throw new IllegalStateException("模拟数据库连接失败：敏感内部信息");
```

`BusinessException` 的处理器使用：

```java
ApiResponse.failure(errorCode, e.getMessage());
```

因为这条异常是主动设计的，调用方通常可以展示其中文业务消息。`Exception` 兜底处理器则使用：

```java
ApiResponse.failure(ErrorCode.INTERNAL_ERROR,
        ErrorCode.INTERNAL_ERROR.getMessage());
```

因为未知异常的消息可能来自数据库、第三方库或内部路径，直接返回会给攻击者提供信息。

### 12. 实测记录

#### 12.1 修复前基线（2026-11-02）

执行方式：Java 21 下 `mvn -q compile` 通过，启动 Spring Boot 后用 `curl` 手工请求。

| 场景 | HTTP | 实际响应 | 结果 |
| --- | ---: | --- | --- |
| `GET /api/v1/health` | 200 | `{"code":0,"message":"success","data":{"service":"hireflow-backend","status":"UP"}}`（JSON 对象键顺序不参与语义） | 符合 |
| `GET /api/v1/exception-demo/business-default` | 404 | `{"code":1004,"message":"请求的资源不存在","data":null}` | 符合 |
| `GET /api/v1/exception-demo/business-custom` | 404 | `{"code":1004,"message":"职位不存在","data":null}` | 符合 |
| `POST /api/v1/exception-demo/validation`，`{"name":""}` | 400 | `{"code":1001,"message":"name:名称不能为空","data":null}` | 符合 |
| 同一接口，非法 JSON | 400 | `{"code":1001,"message":"请求体格式错误","data":null}` | 符合 |
| `GET /api/v1/exception-demo/unknown` | 500 | `{"code":5000,"message":"服务器内部错误","data":null}` | 符合；服务端有完整堆栈 |
| 合法 `{"name":"合法名称"}` | 200 | 空响应体 | 符合当前 void 验证接口 |
| 不存在的 URL | 500 | `{"code":5000,"message":"服务器内部错误","data":null}` | **当时代码的旧行为，后续已修复** |
| 用 POST 访问仅支持 GET 的健康接口 | 500 | `{"code":5000,"message":"服务器内部错误","data":null}` | **当时代码的旧行为，后续已修复** |

上表是修复前的历史基线，不能代表当前代码。

#### 12.2 修复后回归（2026-11-03）

执行方式：Java 21 + Spring Boot 3.4.5 下 `mvn -q compile` 通过，启动后用 `curl` 手工请求；这不是自动化测试。

| 场景 | 实际结果 | 结论 |
| --- | --- | --- |
| `GET /api/v1/health` | 200，`code=0` | 通过 |
| `GET /api/v1/exception-demo/business-default` | 404，`code=1004`，默认消息 | 通过 |
| `GET /api/v1/exception-demo/business-custom` | 404，`code=1004`，“职位不存在” | 通过 |
| `POST /api/v1/exception-demo/validation`，`{"name":""}` | 400，`code=1001`，字段消息 | 通过 |
| 同一接口，`{bad json}` | 400，`code=1001`，“请求体格式错误” | 通过 |
| `GET /api/v1/exception-demo/unknown` | 500，`code=5000`，服务端记录完整堆栈 | 通过 |
| 合法 DTO `{"name":"合法名称"}` | 200，空响应体 | 通过（当前验证接口返回 void） |
| `GET /api/v1/does-not-exist` | 404，`code=1004` | 通过 |
| `POST /api/v1/health` | 405，`code=1006` | 通过 |
| 同上，检查响应头 | 包含 `Allow: GET` | 通过 |

注意：这里完成的是编译、启动和手工 HTTP 验证，不是自动化测试通过。当前阶段按宪章暂缓自动化测试。

### 13. 注解速查

| 注解 | 用在哪里 | 作用 |
| --- | --- | --- |
| `@RestController` | `HealthController`、`ExceptionVerificationController` | 声明 REST Controller；返回对象经 Jackson 写成 JSON |
| `@RequestMapping` | Controller 类 | 给类内接口设置统一路径前缀 |
| `@GetMapping` | 方法 | 映射 HTTP GET 请求 |
| `@PostMapping` | 方法 | 映射 HTTP POST 请求 |
| `@RequestBody` | Controller 参数 | 从请求体读 JSON，交给 Jackson 转对象 |
| `@Valid` | Controller 参数 | 触发 DTO 上的 Bean Validation 约束 |
| `@NotBlank` | DTO 字符串字段 | 禁止 null、空串和纯空白字符串 |
| `@RestControllerAdvice` | 全局异常类 | 注册跨 Controller 的异常处理增强，返回值写响应体 |
| `@ExceptionHandler` | 处理方法 | 声明该方法负责处理的异常类型 |
| `@SpringBootApplication` | `HireFlowApplication` | 组合启动配置、自动配置和组件扫描；扫描起点是 `com.hireflow` |
| `@Test` | `HireFlowApplicationTests` | JUnit 标识测试方法；当前阶段暂缓运行自动化测试 |
| `@Getter` | `ApiResponse`、`ErrorCode` | Lombok 生成 getter，供 Java 调用和 Jackson 序列化 |
| `@AllArgsConstructor` | `ApiResponse` | Lombok 生成包含所有字段的构造方法 |
| `@JsonInclude(ALWAYS)` | `ApiResponse` 类 | 即使值为 null 也输出字段，保持 `data: null` |

`@RequestMapping` 和 `@GetMapping` 在类和方法之间会拼接，例如：

```text
类：@RequestMapping("/api/v1")
方法：@GetMapping("/health")
最终：GET /api/v1/health
```

### 14. 常被追问的泛型问题

```java
public class ApiResponse<T> {
    private final int code;
    private final String message;
    private final T data;
}
```

`T` 是类级类型参数，不是某个固定类。声明 `class ApiResponse<T>` 后，可以把它理解成一个“数据槽类型待定”的响应模板：

```java
ApiResponse<String> text = new ApiResponse<>(0, "success", "hello");
ApiResponse<Map<String, String>> health =
        new ApiResponse<>(0, "success", Map.of("status", "UP"));
```

- `ApiResponse<String>`：`data` 的类型被指定为 `String`。
- `ApiResponse<Map<String, String>>`：`data` 的类型被指定为字符串 Map。
- `ApiResponse<Void>`：`data` 只能放 `null`，适合只表示成功或失败、没有业务数据的情况。

`<T>` 没有把类变成“只能存 T”的特殊容器，而是让编译器记录调用者选择的类型。下面的代码会在编译时报错：

```java
ApiResponse<String> response =
        new ApiResponse<>(0, "success", 123);
// 期望 String，实际放入 Integer，编译不通过
```

这叫编译期类型安全：错误在运行前暴露。泛型运行时仍会发生类型擦除，但这不妨碍编译器检查源码中的类型一致性。

静态方法不能直接复用类上的 `T`，因为静态成员不依赖某个具体实例；所以带数据的工厂方法自己声明类型参数：

```java
public static <T> ApiResponse<T> success(T data)
```

方法签名中的第一个 `<T>` 是方法级类型参数，`ApiResponse<T>` 中的 `T` 使用这个参数。调用时编译器根据实参推断：

```java
ApiResponse<Map<String, String>> response =
        ApiResponse.success(Map.of("status", "UP"));
```

也可以显式指定：

```java
ApiResponse<String> response = ApiResponse.<String>success("ok");
```

无数据版本固定返回 `ApiResponse<Void>`：

```java
public static ApiResponse<Void> success()
```

### 15. 面试问答（异常处理部分）

1. 为什么 `BusinessException` 继承 `RuntimeException` 而不是 `Exception`？
2. HTTP 状态码和 JSON 里的业务码有什么区别？
3. `@RestControllerAdvice` 和普通 `@Controller` 有什么区别？
4. 为什么需要 `Exception.class` 兜底处理器，它会不会覆盖更具体的处理器？
5. `ResponseEntity<ApiResponse<Void>>` 中外层和内层分别代表什么？
6. 为什么返回 `Void` 仍能产生完整 JSON 响应？
7. `@RequestBody`、`@Valid`、`@NotBlank` 分别负责什么？
8. 为什么 `MethodArgumentNotValidException` 可以生成字段级错误消息？
9. 为什么不能把数据库异常文本直接返回客户端？
10. `log.error("...", e)` 与拼接 `e.getMessage()` 有什么区别？
11. Spring singleton Bean 为什么可以被多个请求并发调用？
12. 为什么 Logger 用 `private static final`？
13. JSON 反序列化和序列化方向分别是什么？
14. 为什么 `data: null` 仍然需要出现在响应中？
15. 404 和 405 分别在什么情况下出现，为什么不能合并成同一个错误？
16. 为什么 HTTP 405 响应需要 `Allow` 头？
17. 为什么业务码和 HTTP 状态码要分层，而不是只保留一个数字？
18. `ErrorCode`、`ApiResponse`、`ResponseEntity` 各自负责什么？

### 16. 本步关键修正记录

| 问题 | 原因 | 修正 |
| --- | --- | --- |
| 非法 JSON 返回“参数校验失败” | 直接使用 `INVALID_PARAM` 默认消息 | 为该场景传入固定消息“请求体格式错误” |
| 未知异常没有日志 | 只构造响应，没有记录异常对象 | 增加 SLF4J Logger 和 `log.error(..., e)` |
| 未知异常可能泄露内部信息 | 响应直接使用 `e.getMessage()` 会暴露敏感文本 | 响应固定使用 `INTERNAL_ERROR` 的安全消息 |
| 无效 JSON 没有统一 JSON 体 | 之前未覆盖 `HttpMessageNotReadableException` | 增加专属异常处理器 |
| 未知异常曾走 Spring 默认错误结构 | 之前没有兜底 `Exception` 处理器 | 增加兜底处理器和统一 `ApiResponse` |
| 不存在的 URL 曾返回 500 | `NoResourceFoundException` 被宽泛兜底 | 增加专用处理器，返回 404/1004 |
| HTTP 方法错误曾返回 500 | `HttpRequestMethodNotSupportedException` 被宽泛兜底 | 增加专用处理器，返回 405/1006 |
| 405 缺少 `Allow` | 只设置状态和 body，丢失异常携带的协议头 | 通过 `.headers(e.getHeaders())` 保留 `Allow` |
