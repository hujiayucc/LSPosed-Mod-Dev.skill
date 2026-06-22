# 真实项目模式索引

本文件用于低 token 加载真实项目经验。完整细节见 `LSPosed-Mod-Dev.full.knowledge.md` 第 42-49 章。

## 1. 官方 libxposed/example

可信度：官方最高。

可学习点：

- 现代 API 工程结构；
- `XposedModule` 入口；
- `META-INF/xposed/java_init.list`；
- `module.prop`；
- `scope.list`；
- Gradle `compileOnly`；
- Java/Kotlin 示例；
- service 注册和 Remote Preferences 示例。

转化规则：新模块默认以官方 example 为基准。

## 2. ModernXposedApiDemo

可信度：真实项目，中等。

API：100。

可学习点：

- Java 入门示例；
- `@XposedHooker`；
- `@BeforeInvocation` / `@AfterInvocation`；
- `callback.getArgs()`；
- `callback.setResult()`。

限制：偏旧，不代表 API 102 最佳实践。

## 3. Lsposed-module-template

可信度：真实模板，中等。

API：100。

可学习点：

- 最小目录结构；
- `MainModule.java`；
- `java_init.list`；
- `module.prop`；
- `scope.list`；
- Gradle `compileOnly`。

限制：只能作为基础骨架参考，生成新项目时应升级到 API 102 思路。

## 4. DialerLine

可信度：真实项目，高。

API：101。

可学习点：

- Kotlin 入口类简洁；
- `onModuleLoaded()` 记录框架/API/进程；
- `onPackageReady()` 安装 Hook；
- 单目标包 scope；
- `AtomicBoolean` 防重复安装；
- `Entrypoint` + `HookStrategy` + `Guard` + `State` + `Logger` 分层；
- 统一日志前缀；
- 排错文档和手动测试计划。

转化规则：复杂模块应采用分层架构，入口只路由，Hook Strategy 负责具体 Hook。

## 5. ColorOS-Live-Lyrics-Bridge

可信度：真实项目，高。

API：102。

可学习点：

- `targetApiVersion=102`；
- `exceptionMode=protective`；
- 多进程路由；
- SystemUI / system_server 明确分流；
- `hook(method).setId(...).setExceptionMode(...).intercept(...)`；
- DexKit 定位混淆目标；
- Hook 安装锁；
- 详细日志、构建、发布和测试说明。

转化规则：API 102 复杂模块必须重视进程路由、异常保护、Hook ID、日志和回退。

## 6. MiHealth_AmapFix

可信度：真实项目，中等。

API：100/101 + legacy 兼容。

可学习点：

- 现代入口与旧入口并存；
- API 101+ 等待 `onPackageReady()`；
- `getClassLoader()` / `getDefaultClassLoader()` 兼容；
- 使用 `process|package|classLoader` key 防重复安装；
- 找不到类/方法时记录并跳过。

限制：具体业务 Hook 不作为通用模板；只提炼兼容入口经验。

## 7. 项目质量判断

高质量：

- scope 明确；
- 入口简洁；
- Hook 分层；
- 日志可诊断；
- 异常可回退；
- 文档完整；
- 构建可复现；
- 不滥用 system_server；
- 不把旧 API 当现代最佳实践。

低质量：

- 所有逻辑堆入口类；
- 无包名/进程判断；
- 无异常日志；
- scope 过大；
- 找不到类就崩溃；
- 没有测试和排错文档。