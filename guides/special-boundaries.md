# 特殊场景边界矩阵

本文件用于判断 LSPosed 模块开发中的特殊场景是否适合继续实现。遇到 Java / Kotlin / Native、Android 版本差异、LSPosed / libxposed API 版本限制、SystemUI、system_server、Hot Reload 或 Remote Preferences 时，先用本文件收敛前置条件和风险。

## 快速结论

| 场景 | 默认处理 | 需要确认 |
|---|---|---|
| 普通 App Java Hook | 可协助 | 包名、进程、类名、方法签名、目标版本、分析目标 |
| 普通 App Kotlin Hook | 可协助 | 同 Java；额外确认协程、单例、伴生对象或内联函数影响 |
| Remote Preferences | 可协助 | 默认值、配置来源、权限、失败回退、目标进程 |
| Hot Reload | 可协助，按状态选择 | HookHandle、全局状态、缓存、线程、清理策略 |
| Native Hook | 可协助，先给观测和回退路径 | so、符号、ABI、函数签名、加载时机、崩溃回退 |
| SystemUI Hook | 高影响，先确认 | Android/ROM 版本、分析或修改目标、最小 Hook 点、恢复方案 |
| system_server Hook | 极高影响，先确认 | Android/ROM 版本、目标路径、最小 Hook 点、回滚和救援方案 |

## Java 边界

适合：

- 目标类和方法签名可由源码、反编译代码、运行日志或调用栈确认；
- Hook 点在普通 App 进程内；
- 可以通过 `chain.proceed()` 回退原逻辑；
- 失败时只影响目标 App，不影响系统核心进程。

缺少目标类、方法签名、进程或版本证据时，先输出候选定位、日志观测和最小 Hook 骨架；需要 system_server、SystemUI 或 native 时，同时列出加载时机、回退和恢复信息。

实现要求：

- `libxposed:api` 使用 `compileOnly`；
- 入口先判断 package 和进程；
- 找不到类或方法时记录并跳过；
- Hook 默认使用 `ExceptionMode.DEFAULT`，由 `module.prop` 选择 protective 或 passthrough；
- 命中 Hook 时记录 event、package、process、hook_id、decision。

## Kotlin 边界

Kotlin 模块可以作为默认选择之一，但需要额外注意：

- `object`、伴生对象、顶层函数会改变 JVM 类名和方法名；
- inline 函数、suspend 函数、默认参数会改变实际字节码签名；
- 协程和异步回调中的 Hook 逻辑必须避免阻塞；
- `AtomicBoolean` 或同步块用于防重复安装；
- 不要假设源码层方法名等于运行时方法签名。

生成 Kotlin Hook 前优先核对：

```text
目标 JVM 类名
目标 JVM 方法签名
是否为 suspend / inline / companion / object / top-level
目标进程
失败回退方式
```

## Native Hook 与 Native 分析边界

Native 分析或 Hook 需要明确目标 so、符号或可定位函数、ABI、函数签名、so 加载时机、`native_init.list` 配置、分析目标和崩溃回退策略。

可以协助：

- 检查 `native_init.list`、so 打包路径和 ABI；
- 分析 JNI 注册、导出/隐藏符号、调用关系和加载顺序；
- 设计 Java 层加载日志和失败回退；
- 说明 native 初始化顺序；
- 给出最小验证路径和 native/Java 对照日志。

如果 so、符号、ABI、函数签名或回退信息缺失，先给出 APK so 枚举、JNI 注册检查、加载顺序日志和 Java fallback；系统关键进程另按版本、影响和恢复信息拆分路径。

## Android 版本边界

不同 Android 版本会影响类加载、权限、进程生命周期、存储和系统组件行为。

回答前应确认：

```text
Android 版本
ROM / 厂商定制
目标 App 版本
LSPosed 版本
libxposed API 版本
目标进程列表
是否有源码或反编译依据
```

常见限制：

- 高版本 Android 对非 SDK 接口、隐藏 API、后台行为和文件访问更严格；
- 厂商 ROM 可能修改 SystemUI、权限管理和服务生命周期；
- 同一包名在不同 Android 版本中类名、方法签名和加载时机可能变化；
- system_server 崩溃会影响系统稳定，必须按极高风险处理。

## LSPosed / libxposed API 版本边界

默认目标：`targetApiVersion=102`。

推荐：

```properties
minApiVersion=101
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

边界规则：

- 只支持 API 102 时可设置 `minApiVersion=102`；
- 需要兼容 API 101 时保留 `minApiVersion=101`；
- 旧 API 100/101 项目只能作为迁移参考；
- 不确定 API 名称时必须查分片或官方 API，不伪造方法；
- 新项目不默认使用 `IXposedHookLoadPackage`、`XposedHelpers`、`XSharedPreferences`。

## SystemUI 边界

SystemUI Hook 属于高影响场景，但可用于版本差异分析、兼容适配、日志观测和最小行为修复。继续前确认：

- 目标 Android / ROM / SystemUI 版本明确；
- scope 中确实需要 `com.android.systemui`；
- 目标类、方法签名和调用路径有源码、反编译代码或运行日志依据；
- Hook 点最小、可回退、可关闭；
- 有恢复方案，例如禁用模块、移除 scope、进入安全模式或恢复备份。

推荐输出：

```text
结论：这是 SystemUI 高影响场景，先给出版本、证据和恢复方案。
需要确认：Android/ROM/SystemUI 版本、分析或修改目标、Hook 点、scope、回退方式。
可先做：APK/DEX 静态定位、最小日志模块、只读状态观测、失败不修改行为。
验证：模块加载、类加载、Hook 注册、调用路径、异常和恢复步骤。
```

## system_server 边界

system_server Hook 属于极高影响场景，默认先做静态定位、日志观测和设计审查。进入实现前应同时具备：

- Android/ROM 版本和目标服务明确；
- 目标 Hook 点、调用链和必要性有代码或运行证据；
- 有完整日志、回滚和救援方案；
- 有最小 scope、异常保护、耗时约束和进程恢复步骤。

可协助内容：

- 风险评估和调用链分析；
- 最小 scope 说明；
- 日志和验证计划；
- 如何先在普通 App 或 SystemUI 之外验证思路；
- 如何设计失败时不修改系统行为的保护条件；
- 在证据充分后给出最小可回退 Hook 骨架。

## Remote Preferences 边界

Remote Preferences 用于配置读取；配置值、目标行为和 Hook 结果分别记录，普通配置变化不等同于 Hot Reload。

- 有安全默认值；
- 配置读取失败时继续走原逻辑或默认逻辑；
- 配置值做类型和范围校验；
- 日志记录配置来源：default、remote、invalid、unavailable；
- 不把普通配置刷新误写成 Hot Reload。

## Hot Reload 边界

Hot Reload 只适合能明确管理状态的模块。

默认：

- `autoHotReload=false`；
- 不为普通配置同步启用 Hot Reload；
- 无 HookHandle、状态清理和替换策略时，先输出状态清单、替换方案和重启回退路径，再决定是否生成热重载实现；

启用前优先核对：

```text
全部 HookHandle
全局缓存
后台线程或定时器
配置读取路径
onHotReloading 清理策略
onHotReloaded 替换策略
reload 失败后是否要求重启目标进程
```

## 回答决策模板

遇到特殊场景时，按以下格式回答：

```text
结论：分析 / 设计 / 实现 / 暂缓补证据
影响级别：普通 / 中 / 高 / 极高
输入证据：APK/DEX/smali/so、版本、日志、调用链、复现步骤
需要补充：Android 版本、LSPosed/API 版本、目标包、进程、Hook 点、回退方案
推荐路径：先做静态定位和最小日志，再做最小 Hook，再扩展特殊能力
稳定性：scope、加载时机、异常模式、崩溃保护、恢复步骤
验证：scope、日志、强停/重启、失败回退、禁用模块恢复
```

## 检索建议

- 普通 Java/Kotlin Hook：`knowledge/02-hook-api.md` + Java/Kotlin 模板；
- Remote Preferences / Hot Reload：`knowledge/03-service-remote-hot-reload.md` + `guides/stability-strategy.md`；
- Native Hook：`knowledge/04-native-migration-helper.md` + `cases/api102-real-cases.md`；
- SystemUI / system_server：本文件 + `guides/faq-anti-patterns.md` + `guides/advanced-combinations.md`；
- API 102 复杂模块：`cases/api102-real-cases.md` + `knowledge/06-cases-templates.md`。