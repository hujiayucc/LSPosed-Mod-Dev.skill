# LSPosed-Mod-Dev.skill 导入说明

这是一个面向 **LSPosed 模块开发** 的低 Token Skill 包。它保留完整知识，但通过“启动版 + 按需知识库 + 模板 + 案例索引”的结构降低常驻上下文占用。

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

如果只是想使用这个 Skill，不需要先阅读完整知识库。直接向 AI 助手说明目标即可，例如：

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
| `LSPosed-Mod-Dev.full.knowledge.md` | 完整知识库 | 只按需检索，不全量常驻 |
| `templates/java-api102.md` | Java API 102 模板 | 生成 Java 模块时读取 |
| `templates/kotlin-api102.md` | Kotlin API 102 模板 | 生成 Kotlin 模块时读取 |
| `templates/module-files.md` | Gradle 与 `META-INF/xposed` 模板 | 搭建工程时读取 |
| `guides/quick-start.md` | 普通用户入口、提问方式和需提供信息 | 初次使用或不知道如何提问时读取 |
| `guides/faq-anti-patterns.md` | FAQ、常见反模式和边界请求改写 | 高频问题、质量判断或安全改写时读取 |
| `cases/real-project-patterns.md` | 真实项目架构与质量案例索引 | 架构设计、排错、审查时读取 |
| `cases/migration-compat.md` | 旧 API 迁移与兼容案例索引 | 迁移旧模块时读取 |
| `skill.manifest.json` | 机器可读加载清单 | 项目集成或自动化导入时读取 |

## 推荐导入方式

### 1. 常驻提示词

只把下面文件配置为 Skill 主提示词：

```text
SKILL.md
```

不要把完整知识库常驻注入上下文。

### 2. 知识库 / RAG

把下面文件配置为可检索知识库：

```text
LSPosed-Mod-Dev.full.knowledge.md
templates/
cases/
```

推荐按语义检索、章节检索或显式文件读取方式加载。

### 3. 检索优先级

建议顺序：

1. `SKILL.md`：确定角色、边界、工作流和索引；
2. `templates/`：需要生成工程或代码时优先读；
3. `cases/`：需要架构审查、实战排错、旧 API 迁移时读取；
4. `LSPosed-Mod-Dev.full.knowledge.md`：需要 API 细节、完整说明、排错总表时读取。

## 常见任务映射

| 用户任务 | 推荐读取 |
|---|---|
| 不知道如何开始使用 | `guides/quick-start.md` |
| 新建 LSPosed 模块 | `SKILL.md` + `templates/module-files.md` + Java/Kotlin 模板 |
| 写 Hook 代码 | `SKILL.md` + Java/Kotlin 模板 + 完整知识库 Hook 章节 |
| Remote Preferences | 完整知识库 service / Remote Preferences 章节 |
| Hot Reload | 完整知识库 Hot Reload 章节 |
| Native Hook | 完整知识库 Native Hook 章节 |
| 旧模块迁移 | `cases/migration-compat.md` + 完整知识库旧 API 章节 |
| 模块不生效 | 完整知识库排错章节 + `cases/real-project-patterns.md` |
| 高频问题 / 反模式判断 | `guides/faq-anti-patterns.md` |
| 架构审查 | `cases/real-project-patterns.md` + 完整知识库质量审查章节 |

## Token 策略

- 常驻只加载 `SKILL.md`；
- 完整知识库永远不要全量常驻；
- 模板只在生成代码时加载；
- 案例只在需要架构推理、排错或迁移时加载；
- 如果平台会自动索引目录下所有 Markdown，可以索引全部文件，但仍只把 `SKILL.md` 作为启动提示词。

## 安全策略

本 Skill 只用于合法授权的 LSPosed 模块开发、调试、兼容适配、学习和代码审查。

必须拒绝：

- 绕过检测；
- 反作弊对抗；
- 隐蔽注入；
- 未授权修改第三方 App；
- 窃取隐私或凭据；
- 恶意控制设备；
- 破坏系统稳定；
- 持久化后门。

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