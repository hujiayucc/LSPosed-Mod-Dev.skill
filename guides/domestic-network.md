# 国内网络与依赖获取指南

本文件用于处理国内网络环境下 GitHub、Maven、Gradle 和 libxposed 依赖访问不稳定的问题。它不替代官方来源判断；生成项目或排错时，应优先保持官方依赖坐标和可验证来源，再按网络环境选择镜像、缓存或离线包。

## 适用场景

用户出现以下问题时读取本文件：

- GitHub 页面、release、源码压缩包或 example 仓库访问失败；
- Gradle 解析 `io.github.libxposed:api:102.0.0`、`io.github.libxposed:service:102.0.0` 超时；
- `google()`、`mavenCentral()`、Gradle Plugin Portal 连接慢或间歇失败；
- CI、校园网、企业网、移动网络无法稳定下载依赖；
- 用户只有离线设备，需要先准备依赖缓存和源码参考。

## 基线原则

1. 依赖坐标不因镜像改变而改变，仍以官方坐标为准。
2. 优先使用 Gradle 标准仓库声明，不在模板里硬编码不可验证的第三方 jar。
3. 镜像只作为传输入口，不能替代版本、包名、签名和哈希验证。
4. GitHub 访问失败时，先尝试 release/tag/source archive 或已有离线包，不直接改用来源不明的 fork。
5. 排错时要区分网络失败、仓库缺失、版本不存在、Gradle 配置错误和 Android Gradle Plugin 不兼容。

## Gradle 仓库模板

项目级 `settings.gradle` 或 `settings.gradle.kts` 应集中声明仓库，避免每个 module 分散配置。

```groovy
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
        // 国内网络不稳定时，可在用户确认后加入镜像。
        // maven { url = uri("https://maven.aliyun.com/repository/google") }
        // maven { url = uri("https://maven.aliyun.com/repository/public") }
        // maven { url = uri("https://mirrors.cloud.tencent.com/nexus/repository/maven-public/") }
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // 国内网络不稳定时，可在用户确认后加入镜像。
        // maven { url = uri("https://maven.aliyun.com/repository/google") }
        // maven { url = uri("https://maven.aliyun.com/repository/public") }
        // maven { url = uri("https://mirrors.cloud.tencent.com/nexus/repository/maven-public/") }
    }
}
```

使用规则：

- 能访问官方仓库时，保持 `google()` 和 `mavenCentral()` 优先；
- 国内镜像只在用户明确遇到网络问题时加入；
- 不要删除官方仓库，除非用户环境明确禁止外网；
- 不要把镜像配置写进模块模板的唯一方案，应标注为可选网络策略。

## libxposed 依赖检查

默认依赖仍是：

```groovy
compileOnly("io.github.libxposed:api:102.0.0")
implementation("io.github.libxposed:service:102.0.0") // 仅在需要模块 App 与框架通信时加入
```

检查顺序：

1. Gradle 是否能解析 `io.github.libxposed:api:102.0.0`；
2. 仓库声明是否在 `settings.gradle` 的 `dependencyResolutionManagement` 中生效；
3. 是否被企业代理、校园网认证或 DNS 污染拦截；
4. 镜像仓库是否同步了目标版本；
5. 本地 Gradle 缓存中是否已有该版本；
6. 若仍失败，要求用户提供完整 Gradle 错误，而不是猜测依赖版本。

错误判断：

| 现象 | 常见原因 | 处理 |
|---|---|---|
| `Could not resolve io.github.libxposed:api:102.0.0` | 仓库不可达或镜像未同步 | 切换网络、加入镜像或使用本地缓存 |
| `Could not find io.github.libxposed:api:...` | 版本号错误或仓库缺失 | 回到官方版本和仓库检查 |
| `Read timed out` | 网络慢或代理不稳定 | 增加 Gradle 超时、换网络或使用离线缓存 |
| `PKIX path building failed` | TLS 证书或代理拦截 | 修复 JDK/代理证书，不改依赖坐标 |
| 插件解析失败 | Gradle Plugin Portal 不可达 | 配置插件仓库镜像或使用已缓存插件版本 |

## Gradle 超时和缓存建议

可在用户项目的 `gradle.properties` 中按需加入：

```properties
org.gradle.caching=true
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
systemProp.org.gradle.internal.http.connectionTimeout=60000
systemProp.org.gradle.internal.http.socketTimeout=60000
```

注意：

- 超时只解决慢连接，不解决版本不存在；
- 不建议默认开启离线模式，离线模式会掩盖依赖未缓存的问题；
- CI 可预热 Gradle 缓存，但要记录缓存来源和依赖坐标。

## GitHub 获取失败时的替代路径

官方参考仍以以下项目为基线：

```text
https://github.com/libxposed
https://github.com/libxposed/api
https://github.com/libxposed/service
https://github.com/libxposed/example
https://github.com/libxposed/helper
```

替代获取顺序：

1. 使用 GitHub release、tag 或 source archive 获取固定版本源码；
2. 使用用户已有的离线源码包，并核对项目名、commit、tag 或 release；
3. 使用组织内缓存、制品仓库或 CI 缓存，要求能追溯到官方 commit；
4. 只读参考本 Skill 内的模板和分片，不把模板当作官方完整文档；
5. 不使用来源不明、改过坐标或无法追溯的二进制包。

用户提供离线包时，应要求：

```text
source=GitHub release/tag/archive/local cache
project=libxposed/api|service|example|helper
version_or_commit=<版本或提交>
sha256=<可选哈希>
obtained_at=<获取时间>
```

## 离线开发清单

离线设备或受限网络环境至少准备：

- Android Gradle Plugin 与 Gradle distribution；
- Android SDK platform、build-tools、platform-tools；
- `io.github.libxposed:api:102.0.0`；
- 需要 service 时准备 `io.github.libxposed:service:102.0.0`；
- 项目模板和 `META-INF/xposed` 文件；
- 目标 App 版本、scope、日志和崩溃堆栈；
- 可追溯的 `libxposed/example` 参考源码。

离线回答时不要说“直接同步 Gradle 即可”，应给出两条路径：

```text
在线路径：官方仓库或镜像解析依赖。
离线路径：准备 Gradle cache / Maven local / 内部制品仓库，并核对坐标和版本。
```

## 回答模板

用户说“国内访问 GitHub 或 Maven 很慢”时，建议这样回答：

```text
结论：先保持官方依赖坐标不变，只调整仓库入口和缓存策略。
需要确认：Gradle 错误全文、settings.gradle 仓库配置、是否能访问 Maven Central/GitHub、是否允许使用镜像。
推荐方案：官方仓库优先；失败时加入国内 Maven 镜像；仍失败时使用可追溯的离线缓存。
验证：重新同步 Gradle，确认 libxposed api/service 版本能解析，APK 中仍包含 META-INF/xposed 配置。
风险：不要使用来源不明的 fork、jar 或修改过坐标的依赖。
```

## 与其他文件配合

- 新建模块：`templates/module-files.md`、`knowledge/01-project-basics.md`。
- 依赖和版本验证：`guides/validation-checklist.md`。
- 发布说明和官方基线：`README.md`。
- API 102 代码模板：`templates/java-api102.md`、`templates/kotlin-api102.md`。
