# 实操样例与触发词

本文件用于回答“我该怎么问才能触发对应能力”。它把常见 LSPosed 模块开发任务整理成可直接复制的用户输入、推荐读取路径和回答要点。遇到具体任务时，优先匹配本文件的场景卡，再读取对应模板、指南、案例或知识分片。

## 使用原则

- 用户可以直接描述 APP 逆向、静态分析、动态调试、协议分析、Hook、Instrumentation 或兼容性目标；
- 示例输入可以直接照抄，但应替换目标包名、进程、类名、方法签名、版本信息和输入样本；
- 信息不足时，先列出缺失证据和最小定位步骤，再生成最终 Hook 或分析结论；
- 复杂场景先读 `guides/special-boundaries.md`，确认 SystemUI、system_server、Native、加固、动态加载和高频路径的稳定性要求；
- 代码生成继续遵守最小 scope、结构化日志、错误码和失败回退要求；
- 分析结论必须标注证据来源，并区分事实、推断和待验证假设。

## 触发词速查

| APP 逆向分析 | `分析这个 APK 的 Manifest、DEX、组件、权限和关键调用链，并给出证据表` | `knowledge/01-project-basics.md` + `guides/validation-checklist.md` |
| 静态定位 | `从这段反编译代码/smali 中定位功能入口、调用链和关键参数` | `knowledge/01-project-basics.md` + `knowledge/02-hook-api.md` |
| 动态调试 | `给这个类/方法设计日志 Hook、参数观测和复现路径` | `knowledge/02-hook-api.md` + `guides/troubleshooting-cards.md` |
| 协议与行为分析 | `分析这段请求、序列化数据或 IPC 调用的字段、状态和触发条件` | `knowledge/03-service-remote-hot-reload.md` + `guides/validation-checklist.md` |
| Native / JNI 分析 | `分析这个 so 的 JNI 注册、导出符号、调用关系和加载时机` | `knowledge/04-native-migration-helper.md` + `cases/advanced-native-hook.md` |

| 国内网络 | `GitHub/Maven 下载很慢或 Gradle 解析 libxposed 超时，请给国内镜像和离线缓存方案` | `guides/domestic-network.md` |
| 写 Hook | `我要 Hook ... 类的 ... 方法，签名是 ...，目标进程是 ...` | `knowledge/02-hook-api.md` + Java/Kotlin 模板 |
| 防御性 Hook | `给这个 Hook 加错误码、结构化日志、参数校验和失败回退` | `templates/defensive-error-handling.md` |
| 可靠性 Helper | `给这个复杂 Hook 加 RetryPolicy、TimeoutGuard、FallbackState 和延迟重试 Helper` | `templates/reliability-helpers.md` |
| 模块不生效 | `模块启用后没有日志，这是 module.prop、java_init.list、scope.list 和日志` | `guides/troubleshooting-cards.md` + `guides/validation-checklist.md` |
| 修复前后对比 | `这是错误代码和日志，请给修复前后对比、错误码和验证步骤` | `cases/failure-fix-walkthroughs.md` |
| Hook 不触发 | `module_loaded 有了，但 install_hook/hook_hit 没有，请按链路排查` | `guides/troubleshooting-cards.md` |
| Remote Preferences | `我需要模块 App 给 Hook 进程下发配置，配置项是 ...，默认值是 ...` | `knowledge/03-service-remote-hot-reload.md` |
| Hot Reload | `这个模块是否适合开启 autoHotReload，请按状态和 HookHandle 检查` | `knowledge/03-service-remote-hot-reload.md` + `guides/stability-strategy.md` |
| Native Hook | `我需要 Native Hook，目标 so 是 ...，ABI 是 ...，符号是 ...` | `knowledge/04-native-migration-helper.md` + `guides/special-boundaries.md` + `cases/advanced-native-hook.md` |
| 多模块共存 | `同一目标 App 有多个模块同时启用，请检查 scope 交叉、Hook 优先级和冲突回退` | `guides/multi-module-coexistence.md` |
| API 102 真实案例 | `按 API 102 真实案例帮我设计这个模块结构` | `cases/api102-real-cases.md` |
| 发布前检查 | `请按 V1-V6 验证清单检查这个模块能否发布` | `guides/validation-checklist.md` |
| 架构审查 | `请审查这个模块是否 scope 过大、日志不足或 Hook 点不稳定` | `cases/real-project-patterns.md` + 质量分片 |
| SystemUI / system_server | `这是 SystemUI/system_server 的 APP 兼容、调用链或运行时分析需求，请基于版本和证据给出边界与验证路径` | `guides/special-boundaries.md` |
| 旧 API 迁移 | `把这段 IXposedHookLoadPackage / XposedHelpers 迁移到 API 102` | `cases/migration-compat.md` |

## 场景 1：新建 API 102 Java 模块

用户输入：

```text
帮我新建一个 API 102 的 LSPosed 模块，目标包名是 com.example.app，目标进程是 com.example.app，用 Java。请包含 module.prop、java_init.list、scope.list、入口类和验证步骤。
```

推荐读取：

- `knowledge/01-project-basics.md`；
- `templates/module-files.md`；
- `templates/java-api102.md`；
- 需要防御性日志时再读 `templates/defensive-error-handling.md`。

回答应包含：

```text
结论：可以生成最小 API 102 Java 模块。
关键配置：Gradle、module.prop、java_init.list、scope.list。
代码：ModuleEntry 最小入口、包名/进程判断、module_loaded 日志。
验证：安装、启用 scope、强停目标 App、查看 event=module_loaded。
排错：若无日志，先检查 APK 中 META-INF/xposed 文件。
```

## 场景 2：Kotlin Hook 并带 Guard

用户输入：

```text
我要用 Kotlin Hook com.example.app.UserManager#isVip()，无参数返回 boolean。目标是自有测试 App，进程 com.example.app。请加 installOnce、ExceptionMode.PROTECTIVE、错误码和回退。
```

推荐读取：

- `knowledge/02-hook-api.md`；
- `templates/kotlin-api102.md`；
- `templates/defensive-error-handling.md`。

回答应包含：

```text
结论：根据证据设计最小 Hook 或观测点。
需要确认：目标 App 版本、类是否混淆、真实 JVM 签名、反编译片段和复现路径。
代码：包名/进程 Guard、AtomicBoolean、hook_id、PROTECTIVE、chain.proceed 回退。
验证：module_loaded -> install_hook -> hook_hit，记录参数类型、返回路径和异常。
```

## 场景 3：模块没有任何日志

用户输入：

```text
模块在 LSPosed 里启用了，但目标 App 启动后没有任何模块日志。这是 module.prop、java_init.list、scope.list 和 APK 内容列表，请按步骤排查。
```

推荐读取：

- `guides/troubleshooting-cards.md`；
- `guides/validation-checklist.md`；
- `cases/failure-fix-walkthroughs.md`；
- `knowledge/05-workflow-troubleshooting-quality.md`。

回答应包含：

```text
最可能原因：metadata 未打包、入口类名错误、scope 未命中或 API 版本不匹配。
证据：逐项对照用户给出的文件和 APK 内容。
下一步检查：V2 工程配置 + V3 APK 内容。
修复建议：先修 module.prop/java_init.list/scope.list，再重新安装。
验证：查看 event=module_loaded result=ok。
```

## 场景 4：module_loaded 有但 Hook 不触发

用户输入：

```text
现在能看到 module_loaded，但看不到 install_hook 或 hook_hit。目标类是 com.example.app.FeatureManager，方法 enable(String)，目标进程 com.example.app:remote。请帮我定位。
```

推荐读取：

- `guides/troubleshooting-cards.md`；
- `cases/failure-fix-walkthroughs.md`；
- `templates/defensive-error-handling.md`；
- `knowledge/02-hook-api.md`。

回答应包含：

```text
最可能原因：进程路由、ClassLoader、类名、签名或调用路径不匹配。
检查顺序：scope -> process -> ClassLoader -> class -> method signature -> call path。
建议日志：route_skip、install_hook、class_not_found、method_not_found。
回退：找不到类或方法时跳过该 Hook，不让目标 App 崩溃。
```

## 场景 5：Remote Preferences 配置开关

用户输入：

```text
我需要模块 App 给 Hook 进程下发一个 enable_feature 开关，默认 false，配置读取失败时保持原逻辑。请给出 API 102 方案和排错步骤。
```

推荐读取：

- `knowledge/03-service-remote-hot-reload.md`；
- `cases/api102-real-cases.md`；
- `templates/defensive-error-handling.md`。

回答应包含：

```text
结论：Remote Preferences 适合配置读取，不等同于 Hot Reload。
实现要点：默认值、类型校验、读取失败回退、日志记录配置来源。
错误码：LSM-CFG-001、LSM-CFG-002。
验证：默认配置、远程配置、非法配置、目标进程冷启动分别测试。
```

## 场景 6：Hot Reload 决策

用户输入：

```text
这个模块有 3 个 HookHandle、一个全局缓存和一个后台线程。我想开启 autoHotReload，请判断是否适合，并给出 onHotReloading/onHotReloaded 的验证清单。
```

推荐读取：

- `knowledge/03-service-remote-hot-reload.md`；
- `guides/stability-strategy.md`；
- `guides/validation-checklist.md`。

回答应包含：

```text
结论：默认暂缓，先列出状态和 HookHandle。
需要确认：缓存、线程、HookHandle、配置路径、失败后是否重启目标进程。
实现边界：无清理策略时保持 autoHotReload=false。
验证：reload 前后 hook_hit、旧 Hook 是否残留、失败是否回到重启路径。
```

## 场景 7：Native Hook 信息完整

用户输入：

```text
APP Native 分析需要目标 so、ABI、符号或可定位函数、函数签名、加载时机和日志。请给 native_init.list、Java fallback、错误码和验证路径。
```

推荐读取：

- `guides/special-boundaries.md`；
- `knowledge/04-native-migration-helper.md`；
- `cases/api102-real-cases.md`。

回答应包含：

```text
结论：信息完整，可给最小 Native 分析与可回退骨架。
需要确认：so 打包路径、ABI、符号导出或注册方式、加载时机、Java fallback 和崩溃日志。
验证：native_init.list、APK so、ABI、符号、native_install、hook_hit、Java fallback、异常恢复。
```

## 场景 8：API 102 真实案例转架构

用户输入：

```text
按 API 102 真实案例帮我设计一个模块：普通 App Hook + Remote Preferences + 多进程路由。请给目录结构、核心类职责、日志和验证路径。
```

推荐读取：

- `cases/api102-real-cases.md`；
- `cases/real-project-patterns.md`；
- `knowledge/06-cases-templates.md`；
- `templates/defensive-error-handling.md`。

回答应包含：

```text
架构：ModuleEntry、ProcessRouter、HookStrategy、ConfigStore、Logger。
规则：入口薄、进程先路由、Hook 安装可重复防护、配置默认值和校验。
日志：module_loaded、route_skip、install_hook、hook_hit、read_config、fallback。
验证：每个进程分别启动，检查 install_hook 和配置来源日志。
```

## 场景 9：错误码和防御性代码补强

用户输入：

```text
这是我的 ModuleEntry 代码。请只补错误码、结构化日志、参数校验和失败回退，不改业务逻辑。
```

推荐读取：

- `templates/defensive-error-handling.md`；
- `guides/stability-strategy.md`；
- 对应 Java/Kotlin 模板。

回答应包含：

```text
改动范围：只补 Guard、日志、错误码、catch 分类和 fallback。
不做内容：不改 Hook 目标、不扩大 scope、不改业务返回值。
错误码：按 LOAD/SCOPE/HOOK/CL/SIG/CFG 分类。
验证：触发成功、参数异常、类缺失、签名缺失、配置不可用。
```

## 场景 10：自动化验证 / 发布前检查

用户输入：

```text
请按 V1-V6 验证清单检查这个模块是否可以发布。这是 APK 内容列表、module.prop、java_init.list、scope.list 和 LSPosed 日志。
```

推荐读取：

- `guides/validation-checklist.md`；
- `templates/module-files.md`；
- `templates/defensive-error-handling.md`。

回答应包含：

```text
验证结论：通过 / 不通过 / 信息不足。
检查范围：V1 静态输入、V2 工程配置、V3 APK 内容、V4 运行日志、V5 回退行为、V6 发布前门禁。
失败项：错误码、证据、影响。
修复顺序：配置 -> scope -> 生命周期 -> Hook 签名 -> 回退。
```

## 场景 11：SystemUI 边界判断

用户输入：

```text
这是 SystemUI 的兼容性、调用链或运行时观测需求，目标 Android 版本是 ...，ROM 是 ...，目标类和方法是 ...。请先给出证据、影响级别、最小 Hook 点和恢复方案。
```

推荐读取：

- `guides/special-boundaries.md`；
- `guides/faq-anti-patterns.md`；
- `guides/advanced-combinations.md`。

回答应包含：

```text
结论：可做分析、设计或实现；先按影响级别确定最小路径。
影响级别：高。
需要确认：版本、scope、反编译或日志证据、Hook 点、恢复方案。
推荐路径：先静态定位和只读日志观测，再做最小可回退 Hook。
验证：类加载、Hook 注册、调用路径、异常、禁用模块恢复。
```

## 场景 12：信息不足的按钮 Hook

用户输入：

```text
帮我 Hook 一个按钮。
```

推荐读取：

- `guides/interaction-examples.md`；
- `guides/quick-start.md`；
- 需要定位时再读 Hook API 分片。

回答应包含：

```text
结论：信息不足，先定位再生成可靠 Hook。
需要确认：目标包名、目标进程、Activity/Fragment、按钮点击方法、类名、方法签名、Android/LSPosed/API 版本、反编译片段或运行日志。
可先做：最小日志模块、反编译定位、点击路径日志、候选方法列表。
下一步：提供类名、调用路径或日志后生成 Hook 与验证步骤。
```

## 场景 13：国内网络和依赖解析失败

用户输入：

```text
我在国内网络下同步 Gradle 很慢，GitHub 也经常打不开。libxposed api/service 依赖解析超时，请给 Maven 镜像、离线缓存和验证方案，不要改依赖坐标。
```

推荐读取：

- `guides/domestic-network.md`；
- `templates/module-files.md`；
- `guides/validation-checklist.md`。

回答应包含：

```text
结论：保持官方依赖坐标，只调整仓库入口、超时和缓存策略。
需要确认：Gradle 错误全文、settings.gradle、是否允许使用镜像、是否有离线缓存。
方案：官方仓库优先；失败时加入国内镜像；仍失败时使用可追溯离线缓存。
验证：确认 io.github.libxposed:api:102.0.0 和 service 版本可解析。
```

## 场景 14：多模块共存冲突

用户输入：

```text
同一目标 App 上启用了两个 LSPosed 模块。当前模块只有禁用另一个模块后才生效。请检查 module id、scope 交叉、Hook 优先级、同一方法冲突和回退方案。
```

推荐读取：

- `guides/multi-module-coexistence.md`；
- `guides/advanced-combinations.md`；
- `guides/validation-checklist.md`；
- `templates/defensive-error-handling.md`。

回答应包含：

```text
结论：冲突明确 / 可能冲突 / 信息不足。
证据：scope 交叉、同一 Hook 点、日志顺序、返回值变化或配置键冲突。
修复：命名空间隔离、scope 收窄、Hook 点拆分、参数/返回值 Guard、必要时设置优先级。
验证：单独启用 A、单独启用 B、同时启用 A+B，对比日志。
```

## 场景 15：高级 Native Hook 混合回退

用户输入：

```text
APP Native 分析和混合回退需要目标 so、ABI、符号或可定位函数、函数签名和加载时机。请给 Java fallback、CMake、native_init、native_init.list、错误码和验证路径。
```

推荐读取：

- `guides/special-boundaries.md`；
- `knowledge/04-native-migration-helper.md`；
- `cases/advanced-native-hook.md`；
- `guides/validation-checklist.md`。

回答应包含：

```text
结论：信息完整，可给最小可回退骨架。
输入证据：so、ABI、符号或注册方式、函数签名、加载时机和崩溃日志。
代码范围：Java 入口路由、System.loadLibrary 失败回退、CMake、native_init、符号查找和 backup。
验证：APK so、native_init 导出、ABI、符号、native_install、hook_hit、Java fallback、异常恢复。
```

## 场景 16：程序级重试 / 超时 / 降级 Helper

用户输入：

```text
这个模块有动态 ClassLoader、Remote Preferences 和 Native 分支。请给 RetryPolicy、TimeoutGuard、FallbackState、延迟 Hook 安装 helper，并说明不能在 Hook 回调里长时间阻塞。
```

推荐读取：

- `templates/reliability-helpers.md`；
- `guides/stability-strategy.md`；
- `templates/defensive-error-handling.md`；
- `guides/validation-checklist.md`。

回答应包含：

```text
结论：可以补可靠性 helper，但必须限制重试次数和超时。
代码：RetryPolicy、TimeoutGuard、FallbackState、DelayedHookInstaller。
限制：Hook 回调不 sleep、不无限轮询、不重复调用 chain.proceed。
验证：超时 fallback、重试上限、降级状态日志、Hot Reload 状态清理。
```

## 信息补全模板

当用户没有提供足够信息时，优先按下面格式追问：

```text
还缺这些信息：
1. 目标包名和目标进程
2. Android / LSPosed / libxposed API 版本
3. APK/AAB/DEX/smali/Native so 的版本或哈希
4. 目标类名、方法名、参数和返回值
5. module.prop、java_init.list、scope.list
6. LSPosed 日志、崩溃堆栈、调用链或复现步骤

可以先做的最小步骤：
- 生成只记录 module_loaded 的最小模块；
- 验证 APK 中 META-INF/xposed 文件；
- 启用最小 scope 后强停目标 App；
- 查看 module_loaded / route_skip / install_hook 日志。
```

## 触发判断规则

- 用户提到“怎么问、示例、触发、普通用户、入口”：读取本文件和 `guides/quick-start.md`。
- 用户给出具体输入并问“你会怎么回答”：读取本文件和 `guides/interaction-examples.md`。
- 用户给出 APK/AAB/DEX/smali 并要求静态分析：读取 `knowledge/01-project-basics.md`、`guides/validation-checklist.md` 和 `cases/real-project-patterns.md`。
- 用户给出运行日志、调用栈或动态行为并要求定位：读取 `knowledge/02-hook-api.md` 和 `guides/troubleshooting-cards.md`。
- 用户给出协议、IPC、配置或行为样本：读取 `knowledge/03-service-remote-hot-reload.md` 和 `guides/validation-checklist.md`。
- 用户给出 Native so、JNI 或符号并要求分析：读取 `knowledge/04-native-migration-helper.md`、`cases/advanced-native-hook.md` 和 `guides/special-boundaries.md`。
- 用户给出代码并要求“更稳、错误码、日志、回退”：读取 `templates/defensive-error-handling.md`。
- 用户提到“重试、超时、降级、RetryPolicy、TimeoutGuard、延迟安装”：读取 `templates/reliability-helpers.md` 和 `guides/stability-strategy.md`。
- 用户给出错误代码、崩溃片段、Hook 不触发日志并要求“修复前后对比、真实故障修复过程”：读取 `cases/failure-fix-walkthroughs.md`。
- 用户给出 APK、metadata 或日志并要求“验证、发布前检查”：读取 `guides/validation-checklist.md`。
- 用户提到国内网络、GitHub 打不开、Maven/Gradle 超时、离线依赖：读取 `guides/domestic-network.md`。
- 用户提到多模块共存、Hook 冲突、优先级、scope 交叉：读取 `guides/multi-module-coexistence.md`。
- 用户提到 SystemUI、system_server、Native、加固或动态加载：先读取 `guides/special-boundaries.md`；Native 再读 `cases/advanced-native-hook.md`。