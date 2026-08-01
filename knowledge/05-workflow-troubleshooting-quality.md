# 05 - 工作流、排错与质量审查

覆盖原知识库第 32-41、43-46 章：排错总流程、回答用户的标准流程、Agent 开发能力要求、最小改动原则、常见问题、回答风格、核心原则、高质量架构范式、实战排错、质量审查、Agent 开发工作流增强。

## 排错总流程

当用户说“模块不生效”“Hook 不到”“闪退”“没有日志”“ClassNotFound”时，按以下顺序排查：

```text
环境 -> APK 结构 -> scope -> 生命周期 -> 进程 -> ClassLoader -> 方法签名 -> Hook 触发 -> 异常 -> inline/高级方案
```

不要一开始就改代码。先判断模块是否加载，再判断 scope 是否命中，再判断 Hook 是否有机会触发。

## 环境与 APK 结构

先确认：Android 版本、LSPosed 版本、API 版本、Magisk/Zygisk 状态、模块是否启用、目标包名、目标进程名、是否多用户、模块安装用户、是否重启目标 App、是否需要重启系统。

检查 APK 内是否存在：

```text
META-INF/xposed/java_init.list
META-INF/xposed/module.prop
META-INF/xposed/scope.list
```

确认：`java_init.list` 类名正确；入口类继承 `XposedModule`；入口类有无参构造；混淆规则正确；`module.prop` 合法；`scope.list` 包含目标。

## scope、生命周期、进程、ClassLoader

scope 检查：LSPosed Manager 中模块是否启用；目标 App 是否被勾选；`scope.list` 是否推荐正确；是否多用户；system server 是否使用 `system` scope。

生命周期判断：是否应该用 `onPackageReady()`；是否过早使用目标 ClassLoader；是否应该在 `onSystemServerStarting()`；是否错误地在构造函数中 Hook；是否 `isFirstPackage()` 判断导致跳过。

进程检查：`processName` 是否目标进程；目标类是否只在子进程加载；App 是否多进程；是否 Hook 错进程；是否一个进程加载多个 package。

ClassLoader 检查：是否使用 `param.getClassLoader()`；是否误用模块自己的 ClassLoader；类是否在插件 Dex、Split APK 或延迟加载 Dex；是否动态加载。

## 方法签名、Hook 触发和异常

方法签名检查：方法名、参数类型、参数顺序、基本类型与包装类型、数组、重载、Kotlin 默认参数、suspend 方法、返回类型、方法实际是否在父类。

Hook 触发检查：Hook 是否安装成功；目标方法是否真的被调用；是否 Hook 错重载；类是否已经初始化；是否需要 `hookClassInitializer()`；是否被 inline；必要时才考虑 `deoptimize()`。

异常检查：`ExceptionMode.PROTECTIVE` 是否隐藏了 Hooker 异常；是否应临时改 `PASSTHROUGH`；是否 `chain.proceed()` 抛出目标异常；是否类型转换、参数数组或返回值类型错误。

## 回答用户的标准流程

推荐回答结构：

```text
结论：
- 直接说明推荐方案。

需要确认：
- Android 版本：
- LSPosed/API 版本：
- 模块是否现代 API：
- 目标包名：
- 目标进程：
- 目标类/方法：
- 是否有日志：

实现步骤：
1. 配置文件
2. 入口类
3. 生命周期选择
4. Hook 点选择
5. 验证方式

代码示例：
- 给最小可用代码。
- Java/Kotlin 根据用户项目语言。
- 不确定 API 时明确说明需以当前 Javadoc 为准。

排错清单：
- 入口
- scope
- 进程
- ClassLoader
- 方法签名
- 日志
```

如果用户只问概念，简洁解释即可；如果用户要求写代码，必须给目标文件路径、Gradle 依赖、`META-INF/xposed` 配置、入口类、Hook 安装逻辑、日志、验证方法。

## Agent 能力要求

项目理解：识别现代 API；检查入口、`module.prop`、`scope.list`、Gradle、ProGuard、Manifest；判断是否混用旧 API。

代码生成：能生成 Java/Kotlin 入口、`java_init.list`、`module.prop`、`scope.list`、Gradle 依赖、ProGuard、service 注册、Remote Preferences、Remote Files、Hot Reload 示例、Native Hook 基础模板。

日志分析：查入口是否加载、ClassNotFound、NoSuchMethod、HookFailedError、framework API 不兼容、scope 错误、进程错误、混淆导致的签名错误。

代码审查：检查是否在构造函数中 Hook、是否使用模块 ClassLoader、是否无包名/进程判断、scope 是否过大、target API 102 是否调用 legacy API、是否无日志、是否无异常处理、是否误用 Hot Reload、Remote Preferences 方向是否错误、Native Hook 是否未清理资源。

## 最小改动原则

修改建议应最小化、可验证、不改变无关代码。优先顺序：先让模块能加载，再让 Hook 能触发，再优化架构。不要一次性引入复杂 helper 或 native。

## 常见问题速查

模块没有加载：检查 APK 是否安装、LSPosed 是否启用、`java_init.list` 是否存在、入口类名是否正确、入口类是否继承 `XposedModule`、ProGuard 是否破坏入口、`module.prop` 是否合法、API 版本是否满足。

Hook 不触发：检查 scope、是否重启目标 App、目标进程、`isFirstPackage()` 判断、ClassLoader、方法签名、Hook 时机、目标方法是否被调用、是否 inline。

ClassNotFound：检查是否使用 `param.getClassLoader()`、类名是否混淆、类是否在插件 Dex/Split APK/延迟加载 Dex、是否应该等到 `onPackageReady()`。

NoSuchMethod：检查方法名、参数类型、基本类型与包装类型、数组类型、重载、Kotlin 默认参数、suspend 方法、方法是否在父类、混淆后签名变化。

闪退：检查是否 `PASSTHROUGH`、Hooker 是否抛异常、返回值或参数类型是否错误、`chain.proceed()` 是否调用错误、构造器 Hook 返回是否不当、system_server Hook 是否影响系统稳定。

Remote Preferences 不同步：检查框架是否支持 `PROP_CAP_REMOTE`、App 侧是否拿到 service、Hook 侧是否读取同一 group、listener 是否被 GC、是否误以为 Hook 进程可写、是否应使用 Remote Files。

Hot Reload 不生效：检查 API 是否 >= 102、是否只有一个 Java 入口类、`autoHotReload=true` 或 service 显式触发、`onHotReloading()` 是否返回 true、target 是否来自 `runningTargets`、`data` 是否 classloader-neutral、是否误把配置同步写成 Hot Reload、是否需要在 `onHotReloaded()` 中重装 Hook、是否有旧线程/native 资源未清理。

## 运行时风险与分析分类

按分析影响和稳定性分类：普通 App 静态分析为低风险；普通 App 动态 Hook 和行为观测为中风险；framework 公共 API 观测为中风险；SystemUI、system_server 和 Native Hook 为高风险。高风险路径需要明确版本、加载时机、最小 scope、崩溃保护、日志和恢复方案。

## 输出结构与核心原则

回答时先给分析结论，再列输入证据和缺失信息，再给定位路径、实现方案或补丁，最后给验证与回滚步骤。对于逆向任务，标注 APK/DEX/smali/Native/日志/调用栈等证据来源，区分已验证事实、推断和待验证假设；对于代码任务，明确目标文件、API 版本、ClassLoader、进程、Hook 时机、异常模式和失败回退。

核心原则：现代 API 优先；证据优先于猜测；配置正确优先于写 Hook；scope 和进程必须判断；ClassLoader 是关键；方法签名必须精确；先打日志再优化；Remote Preferences 用于配置；Hot Reload 不用于普通配置同步；Native Hook 仅在明确必要且具备崩溃回退时使用。

复杂模块推荐分层：

```text
app/src/main/java/<package>/
  ModuleEntry.kt 或 ModuleEntry.java
  hook/
    TargetProcessEntrypoint.kt
    XxxHookStrategy.kt
    XxxHookInstaller.kt
  guard/
    SafetyGuard.kt
    PackageGuard.kt
  state/
    ConfigStore.kt
    RuntimeState.kt
  service/
    ModuleServiceBridge.kt
  util/
    Logger.kt
    ReflectionUtils.kt
    ClassLoaderUtils.kt
  model/
    HookTarget.kt
    HookResult.kt
app/src/main/resources/META-INF/xposed/
  java_init.list
  module.prop
  scope.list
```

小模块可以简化，但仍应保留一个入口类、一个 Hook 安装函数、一个 Logger、一个目标包判断、一个异常保护策略。

Hook Installer 职责：定位目标类和方法，设置 Hook ID、优先级、异常模式，注册 intercept，输出成功或失败日志，保证重复调用不会重复安装。

Hook 回调职责：读取参数、检查类型、检查目标条件、必要时读取配置、修改参数或结果、不满足条件时调用原逻辑、所有异常可诊断。不要默认拦截所有调用，不要无判断强转，不要返回错误类型，不要在 system_server 做耗时操作，不要在高频方法输出海量日志。

Guard 常见类型：`PackageGuard`、`ApiGuard`、`ClassGuard`、`MethodGuard`、`ConfigGuard`、`RuntimeGuard`、`EmergencyGuard`、`ThreadGuard`。

Logger 应统一封装模块标签、组件标签、级别、诊断开关、Android Log 与 Xposed log、Throwable 堆栈和结构化字段。

## 实战排错案例

模块安装后完全不加载：优先检查 APK 安装、LSPosed 启用、scope、`module.prop`、`java_init.list`、入口类 public/可加载、混淆、`compileOnly` 是否误写成 `implementation`。

`java_init.list` 指向错误：确认文件内容只有完整类名，包名与源码一致，release 混淆时使用规则，构建产物中 `META-INF/xposed/java_init.list` 被保留。

scope 正确但 Hook 不触发：检查目标是否真的在 scope、是否重启或强停、是否多进程、是否 `isFirstPackage` 跳过、是否应使用 `onPackageReady()`、目标方法是否被调用、是否 inline、目标版本是否签名变化。

ClassLoader 错误：优先用生命周期参数的 `param.getClassLoader()`；API 101+ 普通 App Hook 优先放 `onPackageReady()`；插件类等插件加载后再 Hook；找不到类时记录并跳过。

Hook 后目标 App 闪退：先改成只日志不修改结果，再逐步增加条件，每次只修改一个 Hook 点，不确定时返回 `chain.proceed()`，system_server 问题优先撤回 Hook。

## 质量审查清单

元数据：`module.prop`、`java_init.list`、`scope.list` 或动态 scope、`minApiVersion`、`targetApiVersion`、`staticScope`、`exceptionMode`、`autoHotReload`、APK 中是否误打包 libxposed API。

入口类：继承 `XposedModule`、生命周期正确、记录模块加载、判断包名和进程、避免重复安装、避免入口类堆积复杂逻辑、system_server 单独处理、异常有日志。

Hook 点：最小、稳定、优先公开 framework 路径、避免过度依赖混淆内部类、有版本兼容策略、找不到类/方法有降级、有 Hook ID、异常模式和注册日志。

Hook 回调：参数读取类型安全、条件不满足走原逻辑、返回值类型正确、避免不必要副作用、避免修改共享对象、避免高频日志、避免阻塞 UI 线程、避免 system_server I/O、Throwable 有处理。

日志：能回答模块是否加载、加载在哪个进程、目标包是否命中、Hook 是否注册、Hook 是否触发、为什么跳过、配置是否读取、异常在哪里、当前 API 和框架版本。

文档：README、支持范围、scope 说明、构建方法、安装方法、使用方法、已知限制、排错方法、日志标签、手动测试计划、兼容性说明。

## Agent 开发工作流增强

需求澄清：分析目标、输入样本、哈希或版本、Android 版本、LSPosed/API 版本、目标包名、目标进程、是否有源码、是否有反编译代码或日志、是否需要 UI 配置页、是否需要 Remote Preferences、是否涉及 system_server/SystemUI/native、复现步骤和预期输出。

风险分类：普通 App 静态分析为低到中风险；普通 App 动态 Hook 和行为观测为中风险；Hook framework 公共 API 为中风险；Hook SystemUI、system_server 和 Native Hook 为高风险。风险判断关注加载范围、运行时副作用、数据暴露、崩溃影响和恢复成本。

方案设计必须包含：分析假设、证据来源、API 版本、模块目录结构、元数据配置、生命周期选择、Hook 点理由、ClassLoader 获取方式、失败降级、日志策略、验证步骤、回滚方式。

代码生成必须先最小骨架，再加入一个 Hook 点，再加入日志，再加入配置读取，再加入异常保护；不一次性生成大量未经验证 Hook；不默认生成 system_server、Native Hook 或旧 API 混用。