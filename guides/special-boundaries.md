# 特殊场景边界矩阵

本文件用于判断 LSPosed 模块开发中的特殊场景是否适合继续实现。遇到 Java / Kotlin / Native、Android 版本差异、LSPosed / libxposed API 版本限制、SystemUI、system_server、Hot Reload 或 Remote Preferences 时，先用本文件收敛前置条件和风险。

## 快速结论

| 场景 | 默认处理 | 必须确认 |
|---|---|---|
| 普通 App Java Hook | 可协助 | 授权范围、包名、进程、类名、方法签名、目标版本 |
| 普通 App Kotlin Hook | 可协助 | 同 Java；额外确认协程、单例、伴生对象或内联函数影响 |
| Remote Preferences | 可协助 | 默认值、配置来源、权限、失败回退、目标进程 |
| Hot Reload | 默认暂缓 | HookHandle、全局状态、缓存、线程、清理策略 |
| Native Hook | 默认暂缓 | so、符号、ABI、函数签名、加载时机、崩溃回退 |
| SystemUI Hook | 高风险，先确认 | 合法目的、目标 Android/ROM、最小 Hook 点、恢复方案 |
| system_server Hook | 极高风险，默认不生成 | 合法授权、测试环境、最小 Hook 点、回滚和救援方案 |

## Java 边界

适合：

- 目标 App 为自有、测试、授权分析或兼容适配环境；
- 目标类和方法签名可确认；
- Hook 点在普通 App 进程内；
- 可以通过 `chain.proceed()` 回退原逻辑；
- 失败时只影响目标 App，不影响系统核心进程。

不适合直接生成：

- 未提供目标包名、进程、类名或方法签名；
- 只描述“让某功能失效”“绕过检测”“隐藏模块”；
- 需要 Hook system_server、SystemUI 或 native，但没有说明必要性；
- 目标 App 版本、混淆状态和授权范围不清楚。

实现要求：

- `libxposed:api` 使用 `compileOnly`；
- 入口先判断 package 和 process；
- 找不到类或方法时记录并跳过；
- Hook 使用 `ExceptionMode.PROTECTIVE`；
- 命中 Hook 时记录 event、package、process、hook_id、decision。

## Kotlin 边界

Kotlin 模块可以作为默认选择之一，但需要额外注意：

- `object`、伴生对象、顶层函数会改变 JVM 类名和方法名；
- inline 函数、suspend 函数、默认参数会改变实际字节码签名；
- 协程和异步回调中的 Hook 逻辑必须避免阻塞；
- `AtomicBoolean` 或同步块用于防重复安装；
- 不要假设源码层方法名等于运行时方法签名。

生成 Kotlin Hook 前必须确认：

```text
目标 JVM 类名
目标 JVM 方法签名
是否为 suspend / inline / companion / object / top-level
目标进程
失败回退方式
```

## Native Hook 边界

Native Hook 不是默认方案。只有 Java 层无法覆盖目标逻辑，且信息完整时才进入设计。

必须提供：

- 目标 so；
- 目标符号或可定位函数；
- ABI；
- 函数签名；
- so 加载时机；
- `native_init.list` 配置；
- 崩溃回退策略；
- 授权目的。

可以协助：

- 检查 `native_init.list`、so 打包路径和 ABI；
- 设计 Java 层加载日志和失败回退；
- 说明 native 初始化顺序；
- 在授权测试环境中给出最小验证路径。

必须拒绝或暂缓：

- 目标是绕过检测、反作弊、隐藏注入或规避安全机制；
- 缺少符号、ABI、函数签名或回退策略；
- 要求默认注入系统关键进程；
- 希望隐藏崩溃、隐藏模块或规避审计。

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

SystemUI Hook 属于高风险场景。

可以继续的前置条件：

- 用户明确说明合法授权、测试环境或自有 ROM / 设备；
- 目标 Android / ROM / SystemUI 版本明确；
- scope 中确实需要 `com.android.systemui`；
- Hook 点最小、可回退、可关闭；
- 有恢复方案，例如禁用模块、移除 scope、进入安全模式或恢复备份。

必须拒绝或改写：

- 目标是绕过锁屏、安全提示、支付、权限提示、风控或系统审计；
- 用户要求隐藏模块、隐藏通知、隐藏注入或规避检测；
- 没有恢复方案但要求直接生成高风险 Hook。

推荐输出：

```text
结论：这是 SystemUI 高风险场景，先确认授权和恢复方案。
需要确认：Android/ROM/SystemUI 版本、目标 Hook 点、scope、回退方式。
可先做：最小日志模块、只读状态观测、失败不修改行为。
暂不做：绕过安全机制、隐藏注入、无回退 Hook。
```

## system_server 边界

system_server Hook 属于极高风险场景，默认不生成实现代码。

只有同时满足以下条件才可继续做设计层建议：

- 合法授权明确；
- 测试设备或可恢复环境明确；
- 目标 Hook 点和 Android 版本明确；
- 有完整日志、回滚和救援方案；
- 目标不是绕过安全、隐藏、持久化、规避审计或破坏系统稳定。

可协助内容：

- 风险评估；
- 最小 scope 说明；
- 日志和验证计划；
- 如何先在普通 App 或 SystemUI 之外验证思路；
- 如何设计失败时不修改系统行为的保护条件。

不可协助内容：

- 绕过系统安全策略；
- 禁用安全检查、权限、风控、审计或防护；
- 持久化后门、隐藏模块、隐藏注入；
- 没有回退方案的可执行 Hook 代码。

## Remote Preferences 边界

Remote Preferences 用于配置读取，不用于规避权限或绕过目标 App 逻辑。

必须满足：

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
- 无 HookHandle、状态清理和替换策略时不生成热重载实现。

启用前必须确认：

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
结论：可做 / 暂缓 / 拒绝危险目标
风险级别：普通 / 中 / 高 / 极高
需要确认：授权、Android 版本、LSPosed/API 版本、目标包、进程、Hook 点、回退方案
推荐路径：先做最小可加载模块，再做最小 Hook，再扩展特殊能力
不做内容：列出拒绝或暂缓的危险实现
验证：scope、日志、强停/重启、失败回退、禁用模块恢复
```

## 检索建议

- 普通 Java/Kotlin Hook：`knowledge/02-hook-api.md` + Java/Kotlin 模板；
- Remote Preferences / Hot Reload：`knowledge/03-service-remote-hot-reload.md` + `guides/stability-strategy.md`；
- Native Hook：`knowledge/04-native-migration-helper.md` + `cases/api102-real-cases.md`；
- SystemUI / system_server：本文件 + `guides/faq-anti-patterns.md` + `guides/advanced-combinations.md`；
- API 102 复杂模块：`cases/api102-real-cases.md` + `knowledge/06-cases-templates.md`。