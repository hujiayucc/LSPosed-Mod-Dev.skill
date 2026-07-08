# 自动化验证清单

本文件用于把 LSPosed 模块生成、排错和发布前检查整理成可重复执行的验证路径。它不替代真实设备测试；它给出静态检查、APK 内容检查、运行日志检查和失败验收格式，便于 AI 回答或审查时输出一致的验证步骤。

## 验证分层

| 层级 | 目标 | 失败处理 |
|---|---|---|
| V1 静态输入 | 确认目标、版本、scope、API 和风险边界 | 信息不足时先提问，不生成最终 Hook |
| V2 工程配置 | 确认 Gradle、依赖、metadata 和入口文件 | 先修配置，不排查 Hook 逻辑 |
| V3 APK 内容 | 确认 `META-INF/xposed` 和 native 文件被打包 | 缺文件时重新配置打包 |
| V4 运行日志 | 确认模块加载、路由、Hook 安装和命中 | 按错误码定位失败层级 |
| V5 回退行为 | 确认失败时不破坏目标 App | 修改 Guard、默认值或降级路径 |
| V6 发布前门禁 | 确认最小 scope、日志、排错步骤和风险提示 | 阻止发布或要求补齐文档 |

## V1 静态输入检查

生成代码或给出修复前，至少确认：

```text
target_package=<目标包名>
target_process=<目标进程>
android_version=<Android 版本>
lsposed_version=<LSPosed 版本>
libxposed_api=<目标 API 版本>
scope=<scope.list 内容>
hook_target=<类名/方法/签名>
permission=<授权范围或测试环境>
fallback=<失败回退方式>
```

检查规则：

- 缺少包名、进程、类名或方法签名时，不生成最终 Hook；
- `system`、`android`、`com.android.systemui` 或 system_server 相关需求必须先读 `guides/special-boundaries.md`；
- Native Hook 必须确认 so、ABI、符号、函数签名和回退策略；
- Hot Reload 必须确认 HookHandle、状态清理和 reload 失败路径；
- Remote Preferences 必须确认默认值、类型和值域。

## V2 工程配置检查

必须确认：

```text
compileOnly io.github.libxposed:api:102.0.0
需要模块 App 与框架通信时使用 implementation io.github.libxposed:service:102.0.0
minApiVersion / targetApiVersion 与 module.prop 一致
java_init.list 指向唯一入口类
scope.list 只包含必要包名
入口类继承 XposedModule
```

常见失败：

| 检查项 | 失败现象 | 错误码 |
|---|---|---|
| `module.prop` 缺失 | 模块不可识别 | LSM-VALIDATE-001 |
| `java_init.list` 缺失或入口类错误 | 模块不加载 | LSM-LOAD-002 |
| scope 缺目标包 | Hook 不触发 | LSM-SCOPE-001 |
| scope 过大 | 高风险、难排错 | LSM-SCOPE-002 |
| API 版本过低 | API 不存在或行为不一致 | LSM-API-001 |

## V3 APK 内容检查

构建后检查 APK 内必须存在：

```text
META-INF/xposed/module.prop
META-INF/xposed/java_init.list
META-INF/xposed/scope.list
```

Native Hook 还必须存在：

```text
META-INF/xposed/native_init.list
lib/<abi>/lib<name>.so
```

AI 回答中可以给出以下检查方式，但不要假设用户环境一定有对应命令：

```text
检查 APK 内容：列出 APK 内 META-INF/xposed/ 文件
检查入口类：确认 java_init.list 中类名与实际包名一致
检查 scope：确认 scope.list 只包含目标包名
检查 native：确认 ABI 目录与设备 ABI 匹配
```

验收结论格式：

```text
V3 APK 内容：通过 / 未通过
缺失文件：...
影响：模块不加载 / scope 不生效 / native 入口不执行
修复：补齐打包配置后重新安装
```

## V4 运行日志检查

最小日志链路：

```text
event=module_loaded result=ok
event=route_skip result=skip reason=package_mismatch|process_mismatch
event=install_hook result=ok target=<class.method>
event=hook_hit result=ok|skip
event=fallback result=ok recover=proceed|default|disable_hook
```

如果没有 `module_loaded`：

- 先检查 APK 内容和入口类；
- 再检查 LSPosed 是否启用模块；
- 再检查 API 版本和模块日志。

如果有 `module_loaded` 但没有 `install_hook`：

- 检查 scope；
- 检查包名和进程路由；
- 检查生命周期是否过早；
- 检查 ClassLoader 和类名。

如果有 `install_hook` 但没有 `hook_hit`：

- 检查 Hook 点是否在实际调用路径上；
- 检查方法重载和签名；
- 检查目标 App 版本和混淆变化；
- 添加只读日志，不先扩大 scope。

## V5 回退行为检查

每个 Hook 点都必须有明确失败回退：

| 失败点 | 验收要求 |
|---|---|
| 包名或进程不匹配 | 只记录 skip，不抛异常 |
| 类不存在 | 跳过该 Hook，模块继续加载 |
| 方法不存在 | 记录签名，跳过修改 |
| 参数类型不匹配 | `chain.proceed()` |
| 配置不可用 | 使用默认值 |
| 配置非法 | 丢弃远程值，使用默认值 |
| Hot Reload 失败 | 保持旧状态或提示重启目标进程 |
| Native 初始化失败 | 禁用 native 分支，保留 Java 分支 |

## V6 发布前门禁

发布或交付前必须回答：

```text
scope 是否最小：是 / 否
是否默认使用 API 102：是 / 否
是否包含 module.prop、java_init.list、scope.list：是 / 否
是否包含结构化日志：是 / 否
是否包含错误码：是 / 否
是否使用 ExceptionMode.PROTECTIVE：是 / 否
是否有失败回退：是 / 否
是否有排错步骤：是 / 否
是否涉及 SystemUI/system_server/native：是 / 否；若是，是否已确认边界
```

不通过时，先给出缺失项，不建议继续扩大 Hook 范围。

## 自动化输出模板

当用户要求“帮我验证”“发布前检查”“自动检查模板”时，按以下结构输出：

```text
验证结论：通过 / 不通过 / 信息不足
检查范围：V1/V2/V3/V4/V5/V6
失败项：列出错误码、证据和影响
修复顺序：先配置，再 scope，再生命周期，再 Hook 签名，再回退
需要用户提供：APK 内容列表、module.prop、java_init.list、scope.list、LSPosed 日志
下一步验证：安装、启用 scope、强停目标 App、触发功能、查看日志
```

## 与其他文件配合

- APK 与 metadata：`templates/module-files.md`。
- 防御性代码和错误码：`templates/defensive-error-handling.md`。
- 修复前后对比和真实故障过程：`cases/failure-fix-walkthroughs.md`。
- 模块不生效和崩溃：`guides/troubleshooting-cards.md`。
- 稳定性与降级：`guides/stability-strategy.md`。
- 特殊场景边界：`guides/special-boundaries.md`。