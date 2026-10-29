# M1：数据库与统一响应基础

- 日期：2026-10-29
- 状态：进行中
- 依据：`Constitution/Mission.md`、`TechStack.md`、`RoadMap.md`

## 一、本阶段完成内容

- 在 MySQL 中创建 `hireflow` 和 `hireflow_test` 数据库。
- 在 `hireflow` 中创建：
  - `users`
  - `candidate_profiles`
  - `companies`
- 设计并实现后端统一响应类 `ApiResponse<T>`。
- 使用 Java 21 编译通过。
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

- 使用普通 Java class，暂不使用 record 或 Lombok。
- 字段使用 `private final`，创建后不可修改。
- 提供 getter，供 Jackson 序列化和调用方读取。
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

- 类上的 `T` 是实例级类型参数。
- 静态方法不能直接使用类上的 `T`，需要自己声明 `<T>`。
- `new ApiResponse<>` 的钻石操作符会由编译器根据上下文推断类型。
- Java 泛型主要提供编译期类型安全，运行时会发生类型擦除。
- 无参数成功方法固定返回 `ApiResponse<Void>`，避免依赖上下文推断。

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

## 七、遗留事项

- M1 尚未完成统一异常处理。
- 尚未完成 M1 学习笔记的最终验收和提交。
- 当前阶段暂缓自动化测试；后续是否补测需单独决策。
- MySQL 使用本机账号密码，仓库中不记录真实凭据。

## 八、相关提交

待本轮收尾后补充。

## 九、代码实现补充记录

### `ApiResponse` 已实现的约定

- `code`：`int`，成功为 `0`。
- `message`：`String`，成功默认为 `"success"`。
- `data`：泛型 `T`，没有数据时为 `null`。
- 三个字段均为 `private final`，通过全参构造方法初始化。
- 提供 `getCode()`、`getMessage()`、`getData()`。
- 提供 `success(T data)` 和 `success()` 两个静态工厂方法。
- 类上使用 `@JsonInclude(JsonInclude.Include.ALWAYS)`，防止 `data = null` 被全局 Jackson 配置省略。

### 尚未完成

- `failure` 静态工厂方法。
- `ErrorCode` 错误码设计。
- 全局异常处理。
- 第一个返回统一响应的 Controller 手工验证。

