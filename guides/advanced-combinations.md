# 复杂场景组合指引

这个文件用于指导复杂 LSPosed 模块任务如何组合模板、案例和完整知识库。它不替代具体代码模板，只说明遇到多维需求时先读什么、怎么拆分、哪些风险必须先处理。

## 组合原则

- 先确认合法授权范围，再设计 Hook；
- 先跑通最小 Hook，再组合 service、Hot Reload 或 Native Hook；
- 先保证 scope、入口和日志正确，再排查业务逻辑；
- 每次只新增一个复杂能力，避免同时引入多个不确定因素；
- 每个组合场景都必须保留可回退路径。

## 场景 1：多进程 Hook

适用情况：目标 App 有主进程、remote 进程、provider 进程或独立服务进程。

推荐读取：

- Java/Kotlin API 102 模板；
- 完整知识库生命周期、进程判断、Hook 模型章节；
- `guides/troubleshooting-cards.md` 的 Hook 不触发卡片。

拆分步骤：

1. 在入口处记录 package 和 process；
2. 只对目标进程安装目标 Hook；
3. 每个进程独立记录 Hook 安装结果；
4. 避免把只适合主进程的 ClassLoader 用到其他进程；
5. 如果多个进程都需要 Hook，用策略类拆分，而不是堆在入口类中。

最低日志：

```text
event=process_check package=<pkg> process=<process> accepted=<true|false>
event=install_hook process=<process> target=<class#method> result=<ok|fail>
```

## 场景 2：延迟 ClassLoader / 动态加载

适用情况：目标类在插件、动态 dex、分包或后续初始化阶段才出现。

推荐读取：

- 完整知识库 ClassLoader、生命周期和 Hook 时机章节；
- Hook 模型章节；
- 快速排错卡片的 Hook 不触发部分。

拆分步骤：

1. 先确认目标类是否在默认 ClassLoader 中存在；
2. 不存在时，不要立即判定失败；
3. 找到动态加载入口或稳定初始化方法；
4. 在加载点之后再尝试安装目标 Hook；
5. 设置安装状态，避免重复安装；
6. 失败时记录 ClassLoader、类名和异常。

风险点：

- 不能无限轮询；
- 不能在不确定线程中做重逻辑；
- 不能吞掉 `ClassNotFoundException` 而不记录。

## 场景 3：Remote Preferences + Hook 联动

适用情况：模块 App 提供配置，Hook 进程按配置改变行为。

推荐读取：

- 完整知识库 service、Remote Preferences 章节；
- Java/Kotlin 模板；
- service 失败排错卡片。

拆分步骤：

1. 先让 Hook 在无远程配置时使用安全默认值；
2. 再接入 `libxposed/service`；
3. 捕获 `ServiceException`；
4. 处理 `UnsupportedOperationException`；
5. 配置读取失败时回退默认值；
6. 日志记录配置来源：默认值、远程值或失败回退。

最低策略：

```text
remote_config = success -> use remote value
remote_config = unavailable -> use default value
remote_config = invalid -> ignore and log
```

## 场景 4：Hot Reload + 状态保护

适用情况：模块需要在不重启目标进程的情况下更新 Hook 或配置。

推荐读取：

- 完整知识库 Hot Reload 章节；
- 快速排错卡片 Hot Reload 部分；
- 交互样例 Hot Reload 部分。

拆分步骤：

1. 默认保持 `autoHotReload=false`；
2. 列出所有 HookHandle 和全局状态；
3. 在 `onHotReloading()` 中冻结或拒绝不安全状态；
4. 在 `onHotReloaded()` 中替换 Hook 或清理旧状态；
5. 对仍需保留的 Hook 使用 `HookHandle.replaceHook(...)`；
6. 验证 reload 后旧 Hook 不残留。

风险点：

- 回调可能在 Binder 线程执行；
- 热重载过程中目标进程可能退出；
- 全局缓存和旧 Hook 最容易残留。

## 场景 5：Native Hook + Java Hook 混合模块

适用情况：Java 层无法覆盖目标逻辑，必须同时处理 native 符号。

推荐读取：

- 完整知识库 Native Hook 章节；
- `templates/module-files.md` 的 Native Hook 验证项；
- 快速排错卡片 Native Hook 部分。

拆分步骤：

1. 先完成 Java Hook 最小入口和日志；
2. 再确认 `native_init.list`、so、ABI 和加载路径；
3. 明确目标 so、符号名和函数签名；
4. 在 Java 层记录 `System.loadLibrary()` 成功或失败；
5. 在 native 层记录 `native_init` 版本和初始化结果；
6. Native 失败时 Java 层必须能安全回退。

不要在以下信息缺失时生成 Native Hook：

- 目标 so；
- 目标符号；
- ABI；
- 函数签名；
- 合法授权目的；
- 崩溃回退策略。

## 场景 6：架构审查 + 修复计划

适用情况：用户提供已有模块，希望审查质量或拆分结构。

推荐读取：

- `cases/real-project-patterns.md`；
- `guides/faq-anti-patterns.md`；
- 完整知识库质量审查章节。

审查顺序：

1. scope 是否最小；
2. 入口是否清晰；
3. 生命周期是否正确；
4. Hook 点是否稳定；
5. 日志是否可诊断；
6. 异常是否可回退；
7. service / Hot Reload / Native 是否必要；
8. 是否存在旧 API 惯性写法。

输出建议：

```text
主要问题
风险等级
最小修复步骤
需要暂缓的重构
验证方式
```

## 最小推进策略

复杂需求不要一次性生成完整大模块。优先按以下顺序推进：

1. 最小入口和 metadata；
2. 单个 Hook 点；
3. 日志和异常保护；
4. scope 与进程收敛；
5. 可选 service；
6. 可选 Hot Reload；
7. 可选 Native Hook；
8. 架构清理和复用抽象。