# LSPosed-Mod-Dev.skill 导入说明

这是一个面向 **LSPosed 模块开发** 的低 Token Skill 包。它保留完整知识，但通过“启动版 + 按需知识库 + 模板 + 案例索引”的结构降低常驻上下文占用。

## 文件角色

| 文件 | 角色 | 加载策略 |
|---|---|---|
| `SKILL.md` | Skill 主入口 / 常驻提示词 | 始终加载 |
| `LSPosed-Mod-Dev.full.knowledge.md` | 完整知识库 | 只按需检索，不全量常驻 |
| `templates/java-api102.md` | Java API 102 模板 | 生成 Java 模块时读取 |
| `templates/kotlin-api102.md` | Kotlin API 102 模板 | 生成 Kotlin 模块时读取 |
| `templates/module-files.md` | Gradle 与 `META-INF/xposed` 模板 | 搭建工程时读取 |
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
| 新建 LSPosed 模块 | `SKILL.md` + `templates/module-files.md` + Java/Kotlin 模板 |
| 写 Hook 代码 | `SKILL.md` + Java/Kotlin 模板 + 完整知识库 Hook 章节 |
| Remote Preferences | 完整知识库 service / Remote Preferences 章节 |
| Hot Reload | 完整知识库 Hot Reload 章节 |
| Native Hook | 完整知识库 Native Hook 章节 |
| 旧模块迁移 | `cases/migration-compat.md` + 完整知识库旧 API 章节 |
| 模块不生效 | 完整知识库排错章节 + `cases/real-project-patterns.md` |
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

## 最终推荐

如果你的平台只支持一个 Skill 文件：使用 `SKILL.md`。

如果你的平台支持知识库：使用 `SKILL.md` 作为主入口，并把其他文件作为知识库资源。

如果你的平台支持自动导入清单：优先读取 `skill.manifest.json`。