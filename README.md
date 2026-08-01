# LSPosed-Mod-Dev.skill 导入说明

这是一个面向 Android APP 逆向分析、动态调试、Hook/Instrumentation、兼容适配和 LSPosed 模块开发的低 Token Skill 包。它保留完整知识，但通过“启动版 + 按需知识库 + 模板 + 案例索引”的结构降低常驻上下文占用。

## 官方基线

本 Skill 以 `libxposed` 官方项目作为主要技术基线：

- `libxposed/api`：默认对齐 `io.github.libxposed:api:102.0.0`；
- `libxposed/service`：需要模块 App 与框架通信时使用 `io.github.libxposed:service:102.0.0`；
- `libxposed/example`：作为现代模块目录结构、`META-INF/xposed` 元数据和基础入口写法的优先参考；
- `libxposed/helper`：仅作为复杂查找和混淆定位的可选辅助库，依赖版本生成前必须以 Maven/Gradle 实际可解析版本为准；
- Java 模板使用 `androidx.annotation.NonNull`，项目未包含 AndroidX annotation 时，应额外添加注解依赖或移除模板中的注解 import。

官方参考：

```text
https://github.com/libxposed
https://github.com/libxposed/api
https://github.com/libxposed/service
https://github.com/libxposed/example
https://github.com/libxposed/helper
```

## 普通用户快速开始

如果只是想使用这个 Skill，不需要先阅读知识分片。直接向 AI 助手说明目标即可，例如：

```text
帮我新建一个 API 102 的 LSPosed 模块，目标包名是 com.example.app，用 Java。
```

```text
模块启用后 Hook 没触发，这是 module.prop、scope.list 和 LSPosed 日志，请帮我排查。
```

更多提问方式和需要提供的信息见 `guides/quick-start.md`。

## 文件角色

| 文件 | 角色 | 加载策略 |
|---|---|---|
| `SKILL.md` | Skill 主入口 / 常驻提示词 | 始终加载 |
| `knowledge/index.md` | 知识库分片索引 | 任务映射或不知道读哪个分片时读取 |
| `knowledge/01-project-basics.md` | 工程基础、依赖、元数据、入口类、生命周期 | 新建模块或配置工程时读取 |
| `knowledge/02-hook-api.md` | Hook 链模型、Chain、HookHandle、Invoker、deoptimize | 写 Hook 或解释 API 细节时读取 |
| `knowledge/03-service-remote-hot-reload.md` | Service、Remote Preferences、Remote Files、Hot Reload | 远程配置、框架通信或热重载时读取 |
| `knowledge/04-native-migration-helper.md` | Native Hook、旧 API 迁移、helper、日志规范 | Native、迁移或复杂查找时读取 |
| `knowledge/05-workflow-troubleshooting-quality.md` | 工作流、排错、架构范式和质量审查 | 排错、审查、稳定性分析时读取 |
| `knowledge/06-cases-templates.md` | 真实案例提炼、API 102 模板增强和案例转化规则 | 案例学习、架构设计或模板增强时读取 |
| `knowledge/07-libxposed-api102-reference.md` | 官方 API 102 依赖、元数据、生命周期、Hook 链和 APK-to-Hook 路由 | 核对 API 名称、回调签名、模块元数据或由静态证据生成 Hook 时读取 |
| `templates/java-api102.md` | Java API 102 模板 | 生成 Java 模块时读取 |
| `templates/kotlin-api102.md` | Kotlin API 102 模板 | 生成 Kotlin 模块时读取 |
| `templates/defensive-error-handling.md` | 防御性代码、错误码、结构化日志和失败回退模板 | 生成复杂 Hook、排错或稳定性审查时读取 |
| `templates/reliability-helpers.md` | 程序级重试、超时、降级和延迟安装 Helper 模板 | 生成复杂可靠性代码或稳定性补强时读取 |
| `templates/module-files.md` | Gradle 与 `META-INF/xposed` 模板 | 搭建工程时读取 |
| `guides/quick-start.md` | 普通用户入口、提问方式和需提供信息 | 初次使用或不知道如何提问时读取 |
| `guides/practical-prompts.md` | 可直接复制的实操提问、触发词和场景卡 | 不知道怎么触发能力或需要完整示例时读取 |
| `guides/domestic-network.md` | 国内网络、GitHub、Maven、Gradle 镜像和离线缓存指南 | 依赖解析失败、GitHub 访问慢或离线开发时读取 |
| `guides/multi-module-coexistence.md` | 多模块共存、scope 交叉、Hook 优先级和冲突排查模板 | 同一目标 App 启用多个模块或 Hook 冲突时读取 |
| `guides/faq-anti-patterns.md` | FAQ、常见反模式和逆向分析任务分流 | 高频问题、质量判断或分析路径选择时读取 |
| `guides/troubleshooting-cards.md` | 模块不生效、崩溃和高频失败的一页式排错卡片 | 排错时优先读取 |
| `guides/interaction-examples.md` | 典型输入到推荐回答的样例、边界反馈和信息不足反馈结构 | 不确定回答结构或需要示例时读取 |
| `guides/advanced-combinations.md` | 多进程、延迟 ClassLoader、Remote Preferences、Hot Reload、Native Hook 等复杂组合场景 | 复杂需求拆解或架构审查时读取 |
| `guides/stability-strategy.md` | 重试、超时、参数校验、状态保护和失败降级策略 | 复杂代码生成、排错或稳定性审查时读取 |
| `guides/validation-checklist.md` | 静态输入、APK 内容、运行日志、回退行为和发布前门禁验证清单 | 验证、发布前检查或自动化审查时读取 |
| `cases/real-project-patterns.md` | 真实项目架构与质量案例索引 | 架构设计、排错、审查时读取 |
| `cases/api102-real-cases.md` | API 102 真实案例与场景卡 | 复杂模块、Remote Preferences、Hot Reload、Native Hook 或多进程案例时读取 |
| `cases/advanced-native-hook.md` | 高级 Native Hook、CMake、native_init 和 Java fallback 示例 | Native Hook、JNI、ABI、符号分析或 native 降级设计时读取 |
| `cases/failure-fix-walkthroughs.md` | 真实故障修复过程与修复前后对比 | 模块不生效、Hook 不触发、崩溃或错误代码修复时读取 |
| `cases/migration-compat.md` | 旧 API 迁移与兼容案例索引 | 迁移旧模块时读取 |
| `skill.manifest.json` | 机器可读加载清单 | 项目集成或自动化导入时读取 |

## 推荐导入方式

### 1. 常驻提示词

只把下面文件配置为 Skill 主提示词：

```text
SKILL.md
```

不要把知识分片常驻注入上下文。

### 2. 知识库 / RAG

把下面文件配置为可检索知识库：

```text
knowledge/
templates/
guides/
cases/
```

默认检索 `knowledge/` 分片；需要全文式审计时读取 `knowledge/index.md` 和 7 个主题分片。

### 3. 检索优先级

建议顺序：

1. `SKILL.md`：确定角色、边界、工作流和索引；
2. `guides/`：需要快速入口、实操触发词、FAQ、排错卡片、回答样例、复杂场景拆解或稳定性策略时读取；
3. `knowledge/`：需要 API 细节、工程规则、排错流程、架构质量、全文式审计或案例转化规则时读取对应分片；
4. `templates/`：需要生成工程或代码时读取；
5. `cases/`：需要架构审查、实战排错、旧 API 迁移时读取；

## 常见任务映射

| 用户任务 | 推荐读取 |
|---|---|
| APP 逆向入门 | `SKILL.md` + `knowledge/index.md` + `guides/practical-prompts.md` |
| APK / AAB 静态分析 | `SKILL.md` + `knowledge/01-project-basics.md` + `knowledge/07-libxposed-api102-reference.md` + `guides/validation-checklist.md` |
| APK 到最小观测 Hook | `SKILL.md` + `knowledge/01-project-basics.md` + `knowledge/07-libxposed-api102-reference.md` + `guides/practical-prompts.md` |
| 国内网络 / GitHub / Maven 访问慢 | `guides/domestic-network.md` + `templates/module-files.md` + `guides/validation-checklist.md` |
| DEX / smali / 反编译代码定位 | `knowledge/01-project-basics.md` + `knowledge/02-hook-api.md` + `cases/real-project-patterns.md` |
| 动态调试 / Hook / Instrumentation | `knowledge/02-hook-api.md` + `knowledge/07-libxposed-api102-reference.md` + `guides/troubleshooting-cards.md` + Java/Kotlin 模板 |
| Native / JNI / so 分析 | `knowledge/04-native-migration-helper.md` + `knowledge/07-libxposed-api102-reference.md` + `cases/advanced-native-hook.md` + `guides/advanced-combinations.md` |
| 协议、IPC、配置和行为分析 | `knowledge/03-service-remote-hot-reload.md` + `guides/validation-checklist.md` |
| 新建 LSPosed 模块 | `SKILL.md` + `knowledge/01-project-basics.md` + `knowledge/07-libxposed-api102-reference.md` + `templates/module-files.md` + Java/Kotlin 模板 |
| 写 Hook 代码 | `SKILL.md` + `knowledge/02-hook-api.md` + `knowledge/07-libxposed-api102-reference.md` + Java/Kotlin 模板；复杂场景加 `templates/defensive-error-handling.md` |
| 官方 API 102 名称、枚举、回调或元数据核对 | `knowledge/07-libxposed-api102-reference.md` + 上游 README/Javadoc/example |
| Remote Preferences | `knowledge/03-service-remote-hot-reload.md` |
| Hot Reload | `knowledge/03-service-remote-hot-reload.md` + `guides/stability-strategy.md` |
| Native Hook | `knowledge/04-native-migration-helper.md` + `knowledge/07-libxposed-api102-reference.md` + `guides/advanced-combinations.md` + `cases/advanced-native-hook.md` |
| 多模块共存 / Hook 冲突 | `guides/multi-module-coexistence.md` + `guides/advanced-combinations.md` + `guides/validation-checklist.md` |
| 旧模块迁移 | `cases/migration-compat.md` + `knowledge/04-native-migration-helper.md` |
| 模块不生效 | `guides/troubleshooting-cards.md` + `guides/validation-checklist.md` + `knowledge/05-workflow-troubleshooting-quality.md` + `cases/failure-fix-walkthroughs.md` + `cases/real-project-patterns.md` |
| 高频问题 / 反模式判断 | `guides/faq-anti-patterns.md` |
| 需要交互样例或回答结构 | `guides/interaction-examples.md` + `guides/practical-prompts.md` |
| 错误代码修复前后对比 | `cases/failure-fix-walkthroughs.md` + `guides/troubleshooting-cards.md` + `templates/defensive-error-handling.md` |
| API 102 真实案例 | `knowledge/07-libxposed-api102-reference.md` + `cases/api102-real-cases.md` + `knowledge/06-cases-templates.md` |
| 复杂场景组合 | `guides/advanced-combinations.md` + `cases/api102-real-cases.md` + 对应知识分片或模板 |
| 稳定性审查 / 降级策略 | `guides/stability-strategy.md` + `templates/defensive-error-handling.md` + `templates/reliability-helpers.md` + `guides/validation-checklist.md` + `knowledge/05-workflow-troubleshooting-quality.md` |
| 错误码 / 防御性代码 | `templates/defensive-error-handling.md` + `templates/reliability-helpers.md` + Java/Kotlin 模板 + `guides/stability-strategy.md` |
| 重试 / 超时 / 程序级降级 Helper | `templates/reliability-helpers.md` + `guides/stability-strategy.md` + `templates/defensive-error-handling.md` |
| 自动化验证 / 发布前检查 | `guides/validation-checklist.md` + `templates/module-files.md` + `templates/defensive-error-handling.md` + `templates/reliability-helpers.md` |
| 架构审查 | `cases/real-project-patterns.md` + `cases/api102-real-cases.md` + `knowledge/05-workflow-troubleshooting-quality.md` + `knowledge/06-cases-templates.md` |

## Token 策略

- 常驻只加载 `SKILL.md`；
- 知识分片只按任务读取，不全量常驻；
- 模板只在生成代码时加载；
- 指南只在需要快速入口、实操触发词、FAQ、排错、样例、复杂场景、稳定性策略或验证清单时加载；
- 案例只在需要架构推理、排错或迁移时加载；
- 如果平台会自动索引目录下所有 Markdown，可以索引全部文件，但仍只把 `SKILL.md` 作为启动提示词。

## 分析与工程边界

本 Skill 支持 Android APP 的静态分析、反编译、DEX/smali/Manifest/资源检查、动态调试、Hook/Instrumentation、调用链定位、Native/JNI 分析、协议与行为分析，以及基于分析结果的兼容适配和 LSPosed 模块开发。

分析时保留以下工程要求：输入样本和版本可追溯，结论标注证据来源，运行日志脱敏，Hook 保持最小 scope、可观测和可回退；system_server、SystemUI、Native、加固和高频路径需要额外的崩溃保护、恢复步骤和版本对照。

## 发布前核对

发布到 SkillHub 或 GitHub 前建议确认：

- `README.md`、`SKILL.md`、`skill.manifest.json` 的版本号和发布说明一致；
- 模板中的 `libxposed/api` 与 `libxposed/service` 版本仍能从官方仓库或 Maven 解析；
- `libxposed/helper` 相关依赖只在确需复杂查找时生成，并以实际可解析版本为准；
- Java 模板如保留 `androidx.annotation.NonNull`，目标工程需要具备对应注解依赖；
- GitHub 仓库建议提供 LICENSE、tag 或 release，并在发布说明中关联 SkillHub 页面和对应 commit。

## 许可证

详见 [LICENSE](LICENSE)。

## 最终推荐

如果你的平台只支持一个 Skill 文件：使用 `SKILL.md`。

如果你的平台支持知识库：使用 `SKILL.md` 作为主入口，并把其他文件作为知识库资源。

如果你的平台支持自动导入清单：优先读取 `skill.manifest.json`。