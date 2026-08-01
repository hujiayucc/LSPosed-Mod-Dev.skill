# LSPosed-Mod-Dev 知识库分片索引

本目录把原知识库的 49 个章节按主题拆分为轻量知识节点。默认按任务类型读取本目录文件；需要全文式审计时读取 `knowledge/index.md` 和 7 个主题分片。

## 读取原则

1. 先读 `SKILL.md` 确认角色、能力范围和默认工作流。
2. 再按任务类型读取本目录的 1 到 2 个主题分片。
3. API 名称、回调签名、元数据或生命周期语义有疑问时优先读取 `07-libxposed-api102-reference.md`。
4. 生成工程或代码时再读取 `templates/`。
5. 排错、迁移或架构审查时按需读取 `guides/` 与 `cases/`。
6. 需要全文式审计时读取本索引和全部 7 个主题分片。

## 分片映射

| 分片 | 覆盖原章节 | 适用场景 |
|---|---|---|
| `01-project-basics.md` | 1-13 | 角色、逆向分析范围、依赖、工程结构、元数据、入口类、生命周期 |
| `02-hook-api.md` | 14-21 | Hook 链模型、Chain、优先级、ExceptionMode、HookHandle、Invoker、类初始化器、deoptimize |
| `03-service-remote-hot-reload.md` | 22-27 | Remote Preferences、Remote Files、libxposed/service、scope 请求、Hot Reload |
| `04-native-migration-helper.md` | 28-31 | Native Hook、旧 API 迁移、libxposed/helper、日志规范 |
| `05-workflow-troubleshooting-quality.md` | 32-41、43-46 | 排错流程、回答流程、Agent 能力、架构范式、质量审查、开发工作流 |
| `06-cases-templates.md` | 42、47-49 | 真实项目案例提炼、API 102 模板增强、案例转化规则、最终增强承诺 |
| `07-libxposed-api102-reference.md` | 官方 API 102 补充 | 官方依赖、元数据、生命周期、ClassLoader、Hook 链、Invoker、Hot Reload 与 APK-to-Hook 路由 |

## 常见任务到分片

| 用户任务 | 推荐读取 |
|---|---|
| APP 逆向入门 | `SKILL.md` + `01-project-basics.md` + `guides/practical-prompts.md` |
| APK / AAB 静态分析 | `01-project-basics.md` + `07-libxposed-api102-reference.md` + `guides/validation-checklist.md` |
| DEX / smali / 反编译代码定位 | `01-project-basics.md` + `02-hook-api.md` + `cases/real-project-patterns.md` |
| APK 到最小观测 Hook | `01-project-basics.md` + `07-libxposed-api102-reference.md` + `guides/practical-prompts.md` + `guides/validation-checklist.md` |
| 动态调试 / Hook / Instrumentation | `02-hook-api.md` + `07-libxposed-api102-reference.md` + `guides/troubleshooting-cards.md` |
| 协议、IPC、配置和行为分析 | `03-service-remote-hot-reload.md` + `guides/validation-checklist.md` |
| Native / JNI / so 分析 | `04-native-migration-helper.md` + `07-libxposed-api102-reference.md` + `cases/advanced-native-hook.md` |
| 新建 LSPosed 模块 | `01-project-basics.md` + `07-libxposed-api102-reference.md` + `templates/module-files.md` + Java/Kotlin 模板 |
| 不知道怎么提问或触发能力 | `guides/practical-prompts.md` + `guides/quick-start.md` |
| 国内网络 / GitHub / Maven 访问慢 | `guides/domestic-network.md` + `templates/module-files.md` + `guides/validation-checklist.md` |
| 写 Hook 代码 | `02-hook-api.md` + `07-libxposed-api102-reference.md` + Java/Kotlin 模板；复杂场景加 `templates/defensive-error-handling.md` |
| 官方 API 102 名称、枚举、回调或元数据核对 | `07-libxposed-api102-reference.md` + 上游 README/Javadoc/example |
| Remote Preferences / Remote Files | `03-service-remote-hot-reload.md` |
| Hot Reload | `03-service-remote-hot-reload.md` + `guides/stability-strategy.md` |
| Native Hook | `04-native-migration-helper.md` + `07-libxposed-api102-reference.md` + `guides/advanced-combinations.md` + `cases/advanced-native-hook.md` |
| 多模块共存 / Hook 冲突 | `guides/multi-module-coexistence.md` + `guides/advanced-combinations.md` + `guides/validation-checklist.md` |
| 旧 API 迁移 | `04-native-migration-helper.md` + `cases/migration-compat.md` |
| 模块不生效或崩溃 | `05-workflow-troubleshooting-quality.md` + `guides/troubleshooting-cards.md` + `guides/validation-checklist.md` + `templates/defensive-error-handling.md` + `cases/failure-fix-walkthroughs.md` |
| 错误代码修复前后对比 | `cases/failure-fix-walkthroughs.md` + `guides/troubleshooting-cards.md` + `templates/defensive-error-handling.md` |
| 架构审查 | `05-workflow-troubleshooting-quality.md` + `cases/real-project-patterns.md` + `templates/defensive-error-handling.md` + `guides/validation-checklist.md` |
| 错误码 / 防御性代码 | `templates/defensive-error-handling.md` + `templates/reliability-helpers.md` + `guides/stability-strategy.md` |
| 重试 / 超时 / 程序级降级 Helper | `templates/reliability-helpers.md` + `guides/stability-strategy.md` + `templates/defensive-error-handling.md` |
| 自动化验证 / 发布前检查 | `guides/validation-checklist.md` + `templates/module-files.md` + `templates/defensive-error-handling.md` + `templates/reliability-helpers.md` |
| API 102 真实案例与模板规则 | `07-libxposed-api102-reference.md` + `06-cases-templates.md` + `cases/api102-real-cases.md` + `cases/real-project-patterns.md` |

## 兜底规则

- 分片之间出现冲突时，以官方 `libxposed` 行为和最新分片规则为先。
- 需要全文式审计时，读取本索引和全部 7 个主题分片。
- 对 system_server、SystemUI、Native Hook、加固、动态加载和高频路径，先读取 `SKILL.md` 与对应边界指南，确认版本、证据、加载时机、最小 scope、日志、崩溃保护和恢复步骤。
