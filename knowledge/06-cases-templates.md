# 06 - 真实案例、模板增强与转化规则

覆盖原知识库第 42、47-49 章：真实项目案例库与可复用经验、标准模板增强版、真实案例转化规则、最终增强承诺。

## 案例使用边界

真实项目案例只用于提炼工程结构、API 使用方式、Hook 分层、日志、排错、兼容设计和代码质量模式。不能机械照搬第三方项目业务逻辑。

遇到绕过检测、反作弊、隐藏注入、规避风控、未授权控制第三方应用等项目或需求时，必须拒绝实现细节，只能转为合法学习、调试或兼容性分析。

## 案例学习优先级

1. 官方 `libxposed/example`：现代 API 的最高优先级参考。
2. 官方 `libxposed/api` 与 `libxposed/service`：API 行为、接口边界、生命周期和 service 能力的最终依据。
3. LSPosed Wiki：模块元数据、scope、Native Hook、现代 API 与旧 API 差异的官方说明。
4. 真实现代 API 项目：用于学习实战架构、日志、Hook 安装保护和排错文档。
5. 旧 API 或兼容项目：只用于迁移和兼容参考，不作为现代 API 默认模板。

## 官方 example 核心经验

开发新模块时优先遵循：

- 入口类继承 `io.github.libxposed.api.XposedModule`；
- 入口类写入 `src/main/resources/META-INF/xposed/java_init.list`；
- 模块属性写入 `src/main/resources/META-INF/xposed/module.prop`；
- 静态作用域写入 `src/main/resources/META-INF/xposed/scope.list`；
- `libxposed` API 使用 `compileOnly`，不得打包进 APK；
- `onModuleLoaded()` 记录模块加载、框架名称、API 版本和进程；
- `onPackageLoaded()` 或 `onPackageReady()` 按目标包名和进程安装 Hook；
- 使用 `XposedModule.log()` 或统一 Logger 输出可诊断日志。

## 旧示例的可学点与限制

`gauravssnl/ModernXposedApiDemo` 适合学习基础入口和注解式 Hooker，但 API 版本偏旧，`module.prop` 使用 API 100，不代表 API 102 最佳实践。新项目不能把它作为默认模板，只能作为 Java 入门示例。

`luolong47/Lsposed-module-template` 适合学习最小目录结构，但 API 版本示例不是 API 102。生成新 Skill 代码时应升级为：默认 `targetApiVersion=102`；保守兼容时可设置 `minApiVersion=101`；现代入口优先使用无参构造和 `onModuleLoaded()`；复杂 Hook 优先使用 `hook(method).intercept(...)` 链式 API；迁移场景才考虑旧 `XposedHelpers`。

`ljy6-6-6/MiHealth_AmapFix` 适合学习现代/旧 API 兼容入口和重复安装保护，但不应把具体业务 Hook 作为通用模板。兼容经验包括：API 101+ 尽量在 `onPackageReady()` 安装 Hook；找不到目标类/方法时记录日志并跳过；兼容层必须清楚标注“临时迁移方案”。

## 现代 API 102 项目经验

高质量 Kotlin 现代模块架构可学习：

- 入口类很薄，只负责生命周期、目标包判断和路由；
- `onModuleLoaded()` 中绑定 Logger，并记录 `processName`、`api`、`framework`、`frameworkVersion`；
- `onPackageReady()` 中判断 `packageName` 和 `isFirstPackage`；
- 只对目标包启用 Hook，scope 收敛明确；
- 使用 `installOnce(...)` 或 `AtomicBoolean` 防止重复安装；
- Hook 拆成 ProcessEntrypoint、HookStrategy、Guard、State、UI、util 等模块；
- 优先 Hook Android framework 公开入口，减少对目标 App 混淆内部实现的依赖；
- 对业务条件设置安全门，不满足时直接 `chain.proceed()`；
- 修改参数时复制原对象，避免共享对象副作用；
- 文档包含架构、开发、排错和手动测试计划。

复杂 API 102 项目可学习。需要更完整的复杂模块、Remote Preferences、Hot Reload、Native Hook 和多进程场景时，优先读取 `cases/api102-real-cases.md`，再读取 `guides/advanced-combinations.md` 和对应知识分片。

- `module.prop` 使用 `minApiVersion=102`、`targetApiVersion=102`、`staticScope=true`、`exceptionMode=protective`、`autoHotReload=false`；
- `java_init.list` 指向唯一入口类；
- 多进程模块先写进程路由表，不在所有进程盲目安装 Hook；
- system_server 和 SystemUI Hook 单独列为高风险路径；
- `hook(method).setId(...).setExceptionMode(PROTECTIVE).intercept(...)`；
- 混淆目标可使用 DexKit 或 helper 辅助定位，但不能代替清晰 Hook 条件设计；
- 对每个 Hook 安装点使用布尔锁或同步块防止重复安装；
- `scope.list` 出现 `system` 或 `com.android.systemui` 时必须解释原因和风险；
- 发布前验证 APK 内 `META-INF/xposed` 元数据完整。

## 可转化为通用 Skill 规则

- 新模块入口类不得堆积全部 Hook 逻辑；
- 每类 Hook 应有独立 Strategy 或 Installer；
- 每个 Hook 都应有注册日志、命中日志、跳过原因日志、异常日志；
- 每个 Hook 都应能在条件不满足时安全回退到原始逻辑；
- 每个复杂模块都应提供手动测试计划和排错文档；
- API 102 项目优先使用 `ExceptionMode.PROTECTIVE`；
- 多进程模块必须先写进程路由表；
- DexKit/helper 只能用于定位混淆目标，不能代替清晰的 Hook 条件设计；
- 发布前必须验证 APK 内 `META-INF/xposed` 元数据完整。

## API 102 Java 入口模板要点

模板应具备：

- `ModuleEntry extends XposedModule`；
- `TAG`、`TARGET_PACKAGE` 常量；
- `volatile boolean installed` 或等价重复安装保护；
- `onModuleLoaded()` 记录 `event=module_loaded`、进程、API、框架名和框架版本；
- `onPackageReady()` 判断目标包和 `isFirstPackage()`；
- `installHooks(ClassLoader)` 中同步安装；
- 找类、找方法、`method.setAccessible(true)`；
- `hook(method).setId(...).setExceptionMode(PROTECTIVE).intercept(...)`；
- 参数类型检查，不符合时 `chain.proceed()`；
- 成功、类缺失、方法缺失、安装失败都有日志。

## API 102 Kotlin 入口模板要点

模板应具备：

- `class ModuleEntry : XposedModule()`；
- `AtomicBoolean(false)` 做重复安装保护；
- `onModuleLoaded()` 记录模块加载、进程、API 和框架；
- `onPackageReady()` 判断目标包和 `isFirstPackage`；
- `installHooks(classLoader)` 用 `compareAndSet(false, true)`；
- `classLoader.loadClass(...)` 和 `getDeclaredMethod(...)`；
- `method.isAccessible = true`；
- `setId(...)`、`setExceptionMode(PROTECTIVE)`、`intercept`；
- 参数安全转换，不符合条件走原逻辑；
- 失败时重置安装状态并记录错误。

## module.prop、java_init.list、scope.list 模板

推荐 `module.prop`：

```properties
minApiVersion=101
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

说明：只使用 API 102 且不考虑旧框架时，可设置 `minApiVersion=102`；需要兼容 API 101 时，可设置 `minApiVersion=101`；`autoHotReload=true` 只在明确支持热重载时启用。

`java_init.list`：

```text
com.example.module.ModuleEntry
```

要求：每行一个入口类；API 102 Hot Reload 场景最好只有一个 Java 入口；类名必须完整；不要多余空格；混淆后必须保证文件能被正确重写或入口类不被混淆删除。

`scope.list`：

```text
com.example.target
```

如果需要 SystemUI：

```text
com.example.target
com.android.systemui
```

如果需要 system_server：

```text
system
com.example.target
```

注意：`system` 和 `com.android.systemui` 必须有明确理由；不要为了“保险”扩大 scope；scope 越大，风险越高，排错越复杂。

## 真实案例转化格式

学习真实项目时必须按以下格式总结，不能只堆项目名：

```text
项目：<仓库名>
可信度：官方 / 高 / 中 / 低
API：100 / 101 / 102 / legacy
可学习点：工程结构、入口、Hook 方式、日志、排错、兼容
不可照搬点：具体业务逻辑、敏感 Hook、过时 API、项目特定假设
可转化规则：写成通用开发规范
```

允许纳入：目录结构、Gradle 配置、`META-INF/xposed` 文件配置、生命周期选择、Hook 分层架构、重复安装保护、Logger、Guard、排错文档、手动测试流程、API 迁移经验、兼容策略。

不能纳入：绕过检测实现、反作弊对抗、隐藏模块或隐藏注入、未授权修改第三方服务、盗用或破解、恶意控制设备、窃取隐私、对抗安全软件、具体敏感业务 Hook 代码。

## 项目质量判断标准

高质量项目通常具备：目标范围窄、scope 明确、入口类简洁、Hook 分层清楚、日志可诊断、异常可回退、文档完整、构建可复现、不盲目扩大权限、不滥用 system_server、不把旧 API 当现代最佳实践。

低质量项目常见问题：入口类几千行、所有逻辑堆在一个类、无包名判断、无进程判断、无异常日志、找不到类就崩溃、scope 过大、使用旧 API 但不说明、不提供测试方法、不提供排错说明。

## 最终增强承诺

Skill 必须能从真实项目中提炼通用开发模式，区分官方基准、现代实战、旧 API 迁移和低质量示例；能设计清晰目录结构，把入口、Hook、Guard、Logger、State、Service 分层；能生成 API 102 优先的 Java/Kotlin 模板；能给出可执行排错顺序；能审查 `module.prop`、`java_init.list`、`scope.list`；能识别 Hook 时机、ClassLoader、进程、scope、方法签名问题；能建议最小 Hook 点和安全回退策略；能避免把旧 API 示例误用为现代最佳实践；能拒绝高风险和未授权用途。