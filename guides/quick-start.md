# 普通用户快速入口

这个文件用于帮助普通用户快速开始使用 LSPosed-Mod-Dev Skill。它不替代 `SKILL.md`，只回答“我该怎么问、该提供什么信息、会得到什么结果”。

## 使用方式

你不需要直接阅读知识分片，也不需要一次性说明所有实现细节。把目标告诉 AI 助手即可，助手会按需读取模板、案例和知识分片。

推荐这样提问：

```text
帮我新建一个 API 102 的 LSPosed 模块，目标包名是 com.example.app，用 Java。
```

```text
我的模块启用后 Hook 没触发，请根据日志和配置帮我排查。
```

```text
把这个旧 XposedHelpers 写法迁移到 libxposed API 102。
```

```text
我想给模块增加 Remote Preferences 配置能力。
```

```text
拿到一个 APK，请先分析 Manifest、DEX、组件、进程、关键字符串和 so，再从调用链中选一个方法做 API 102 观测 Hook。请按 APK -> 候选方法 -> ClassLoader/进程 -> 最小 Hook -> 日志验证的顺序输出。
```

```text
这是反编译代码、运行日志和目标功能描述，请定位调用链，给出待验证假设、动态观测点和可直接使用的 Hook 模板。
```

```text
我需要 Native Hook 的最小工程结构和验证步骤。
```

更多可直接复制的提问方式见 `guides/practical-prompts.md`。该文件按 APK 静态分析、动态调试、Hook、Remote Preferences、Hot Reload、Native、错误码和验证整理了实操触发词。

## 提供信息

为了减少来回追问，尽量提供：

- 目标包名和目标进程；
- Android 版本、LSPosed 版本、目标 API 版本；
- 语言偏好：Java 或 Kotlin；
- 目标方法、类名、参数签名或已有代码片段；
- `module.prop`、`java_init.list`、`scope.list` 的当前内容；
- LSPosed 日志、目标 App 崩溃日志或模块日志；
- 分析目标、样本版本或哈希、复现路径和已有证据；

## 常见入口

| 目标 | 推荐提问 |
|---|---|
| 新建模块 | `帮我生成一个 Java API 102 模块骨架，目标包名是 ...` |
| 写 Hook | `我要 Hook ... 方法，请给出入口类和验证步骤` |
| 排查不生效 | `模块不生效，这是配置和日志，请按清单排查` |
| 旧 API 迁移 | `把这段 IXposedHookLoadPackage / XposedHelpers 代码迁移到 API 102` |
| Remote Preferences | `我需要模块 App 给 Hook 进程下发配置，请给出 service 方案` |
| Hot Reload | `这个模块是否适合开启 autoHotReload，请给出实现和风险` |
| Native Hook | `我需要 Native Hook，请给出 native_init 和 ABI 检查项` |
| 架构审查 | `请审查这个模块结构是否 scope 过大、日志不足或 Hook 点不稳定` |

## 输出预期

一次完整协助通常会包含：

- 结论和风险判断；
- 需要确认的信息；
- 推荐读取或使用的模板；
- 关键配置；
- 代码或补丁；
- 验证步骤；
- 排错清单。

## 分析工作流入口

可以直接提交以下材料：APK/AAB、DEX、smali、反编译代码、Native so、Manifest、崩溃堆栈、运行日志、抓包摘要、类名/方法签名、版本信息和复现步骤。Skill 会按需选择静态分析、动态调试、Hook/Instrumentation、调用链、协议或兼容性路径，并输出证据、结论、代码/补丁和验证步骤。

### APK 到最小观测 Hook

```text
1. APK/AAB -> Manifest、组件、进程、DEX、资源、so、JNI 注册
2. 静态调用链 -> 候选类、方法、参数、返回值和加载条件
3. 动态日志 -> package/process/ClassLoader/调用栈/命中次数
4. API 102 -> onPackageReady + 目标 ClassLoader + Hooker Chain
5. 验证链路 -> module_loaded -> route -> install_hook -> hook_hit -> fallback
```

如果类名、方法签名或进程仍不确定，先输出候选列表、待验证假设和只读观测 Hook；资料缺口不阻断普通 APP 逆向分析。
