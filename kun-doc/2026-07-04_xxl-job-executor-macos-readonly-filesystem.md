# xxl-job executor 启动报 Read-only file system（macOS）

> 日期：2026-07-04
> 场景：本地 macOS（Apple Silicon）运行 xxl-job-executor-sample-springboot，Spring 容器启动失败，executor 初始化日志目录时报错
> 环境：macOS 26.5.1（arm64）/ xxl-job 源码版 executor-sample / Spring Boot

## 1. 现象

启动 `XxlJobExecutorApplication` 直接失败退出，关键堆栈：

```
java.lang.RuntimeException: java.nio.file.FileSystemException: /data: Read-only file system
	at com.xxl.job.core.executor.impl.XxlJobSpringExecutor.afterSingletonsInstantiated(XxlJobSpringExecutor.java:62)
	...
Caused by: java.nio.file.FileSystemException: /data: Read-only file system
	at java.base/sun.nio.fs.UnixFileSystemProvider.createDirectory(UnixFileSystemProvider.java:397)
	at java.base/java.nio.file.Files.createDirectories(Files.java:793)
	at com.xxl.tool.io.FileTool.createDirectories(FileTool.java:140)
	at com.xxl.job.core.log.XxlJobFileAppender.initLogPath(XxlJobFileAppender.java:52)
	at com.xxl.job.core.executor.XxlJobExecutor.start(XxlJobExecutor.java:151)
```

特点：不是依赖、端口、数据库问题，而是 NIO 层创建目录被拒。

## 2. 排查过程

| 步骤 | 动作 | 结论 |
|------|------|------|
| 1 | 读堆栈定位触发点：`afterSingletonsInstantiated` → `XxlJobExecutor.start` → `XxlJobFileAppender.initLogPath` | executor 初始化时主动建目录，业务代码无关 |
| 2 | 追到配置源头 `application.properties` 中 `xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler` | xxl-job 不像 Spring 属性那样支持 `:` 默认值写法，路径被原样传给 `Files.createDirectories()` |
| 3 | 手动 `mkdir /data` 验证 → 同样报 `Read-only file system` | 根因确认：macOS SIP 保护，`/data` 根目录不可写，与权限无关 |
| 4 | 对照同项目 `logback.xml`：`value="${LOG_HOME:-/Users/wankun/data/applogs}/..."` 且 file appender 已被注释 | logback 侧已本地化过，但 executor logpath 漏改，两处路径不一致 |

### 根因

`XxlJobFileAppender.initLogPath()` 在 executor 启动时对 `logpath` 逐级建目录（`Files.createDirectories`）。默认配置写死了 Linux 风格的 `/data/applogs/...`，而 macOS 上 `/data` 属于 SIP 保护的只读路径，创建即抛 `FileSystemException`，Spring 容器随之启动失败。

一句话：**配置文件里的绝对路径与运行环境文件系统不匹配**，不是代码 bug。

## 3. 解决方案

把 `logpath` 改到用户可写目录，并与 logback 侧 `${LOG_HOME:-/Users/wankun/data/applogs}` 的基路径保持一致。

### 修改内容

涉及两个示例模块（同一个坑，一起改掉避免下次再踩）：

| 文件 | 修改 |
|------|------|
| `xxl-job-executor-samples/xxl-job-executor-sample-springboot/src/main/resources/application.properties` | `xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler` → `/Users/wankun/data/applogs/xxl-job/jobhandler` |
| `xxl-job-executor-samples/xxl-job-executor-sample-springboot-ai/src/main/resources/application.properties` | 同上 |

```properties
### xxl-job executor log-path
xxl.job.executor.logpath=/Users/wankun/data/applogs/xxl-job/jobhandler
```

### 验证

```bash
# 目录预创建，确认可写
mkdir -p /Users/wankun/data/applogs/xxl-job/jobhandler

# 重启 XxlJobExecutorApplication
# 观察：容器正常起来，executor 注册到 admin（127.0.0.1:8080），9999 端口监听
```

启动成功，无 `FileSystemException`。

## 4. 踩坑点

- xxl-job 的 `logpath` 是 executor 原生配置，**不走 Spring 的属性占位符解析**，不能写 `${LOG_HOME:-xxx}` 这种默认值语法，只能给一个确定的路径
- `/data` 只读是 macOS SIP 行为，`sudo` 也绕不过，别在权限上浪费时间
- `initLogPath` 失败会直接中断 Spring 启动（`afterSingletonsInstantiated` 抛 RuntimeException），日志看起来像 Spring 容器问题，容易被堆栈上层的 `Application run failed` 带偏
- 官方 sample 配置默认面向 Linux 生产环境，macOS 本地跑 sample 前先检查所有绝对路径配置

## 5. 备选方案

如果不想把个人路径硬编码进 git 工作区，也可以：

```properties
# 相对路径，落在项目工作目录下（IDEA 默认是模块根目录）
xxl.job.executor.logpath=./data/applogs/xxl-job/jobhandler
```

或启动时通过命令行覆盖，不动源文件：

```bash
java -jar xxl-job-executor-sample-springboot.jar \
  --xxl.job.executor.logpath=/Users/wankun/data/applogs/xxl-job/jobhandler
```

## 6. Checklist

- [x] 堆栈定位到 `XxlJobFileAppender.initLogPath`，排除业务代码问题
- [x] 确认 `/data` 为 macOS SIP 只读（手动 mkdir 复现）
- [x] 两个 sample 模块的 `logpath` 统一改为可写路径
- [x] 预创建目录 `mkdir -p /Users/wankun/data/applogs/xxl-job/jobhandler`
- [x] 重启 executor 验证通过
