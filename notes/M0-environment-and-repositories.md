# M0：环境与三仓库基线

- 验收日期：2026-10-28
- 状态：DONE

## 可运行结果

- Java 21 后端最小工程可启动，健康接口 `/api/v1/health` 返回 `UP`。
- Vue 3 + TypeScript 前端完成最小路由壳，可通过 Vite 启动。
- 三个独立 Git 仓库在本地和 GitHub 上均可用。

## 验证记录

- 后端：`mvn test`、`mvn package`、实际启动和健康接口验证通过。
- 前端：`npm run type-check`、`npm run build` 通过。
- Git：三个仓库的 `main` 与 `origin/main` 提交一致。
- IDEA：确认 backend、frontend、docs 可分别独立打开。

## 学习笔记

### 为什么使用三个仓库

外层 `HireFlow/` 只是工作区。`backend/`、`frontend/`、`docs/` 各自有 `.git`，所以前后端不仅目录分离，版本历史和 GitHub 远程仓库也分离。

### 为什么检查要分层

编译成功不代表应用能启动；应用能启动也不代表接口响应正确。后端分别验证编译、测试、打包、启动和健康接口，可以更早定位问题。

### 踩坑记录

- 最初只用 `JAVA_HOME` 启动 jar，但 shell 中的 `java` 仍指向 Java 17，导致 `UnsupportedClassVersionError`；改用 Java 21 的绝对路径验证通过。
- 前端 Vite 别名和 TypeScript 别名是两套配置，缺少 `tsconfig` 映射会导致构建失败。

## 面试问题

1. 为什么前后端要拆成独立仓库？
2. Monorepo 和多仓库各有什么取舍？
3. 为什么项目需要健康检查接口？
4. 编译、单元测试、打包和启动验证分别能证明什么？

## 相关提交

- backend：`28ad95f`、`a9c03f7`
- frontend：`e1d9820`
- docs：`c7a50f0`

## 遗留问题

MySQL 账号和建库由学习者本人管理，在需要数据库联调前完成。

## 下一步

进入 M1：设计核心表、SQL 初始化、统一响应与异常处理。
