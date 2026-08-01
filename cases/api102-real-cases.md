# API 102 真实案例与场景卡

本文件补充 API 102 优先的案例材料。它用于把可验证项目经验转化为通用工程规则，不能机械照搬第三方项目的业务逻辑，也不能把未核实仓库包装成“真实案例”。

## 使用边界

- 真实项目名只用于说明工程结构、API 用法、日志、排错和质量模式。
- 未确认来源的内容只能写成“场景卡”或“模板化模式”，不能写成真实仓库案例。
- 涉及检测、反作弊、风控、模块隐藏或第三方 App 修改机制时，先拆分静态定位、动态观测、兼容修复和行为验证，输出可复现的证据链和最小实现范围。
- system_server、SystemUI、Native Hook 先记录版本、影响等级、必要性、回退策略和验证环境，再选择分析、观测或实现路径。

## 案例 1：官方 libxposed/example 基线

来源类型：官方示例。

API 定位：现代 libxposed API 基准，优先作为新模块骨架参考。

可学习点：

- 使用 `XposedModule` 作为入口基类；
- 入口类写入 `META-INF/xposed/java_init.list`；
- `module.prop`、`scope.list` 放在 `META-INF/xposed/`；
- `libxposed` 依赖使用 `compileOnly`，不打包进 APK；
- 在生命周期回调中记录模块加载、目标包、进程、框架和 API 版本；
- service / Remote Preferences 相关能力以官方 `libxposed/service` 行为为准。

可转化规则：

- 新建模块时先按官方 example 生成最小可加载工程，再增加 Hook 逻辑；
- 不先写复杂 Hook；先验证 `module.prop`、`java_init.list`、`scope.list` 是否进入 APK；
- 任何 API 102 模板都必须默认包含可诊断日志和 scope 收敛。

推荐输出结构：

```text
结论：先生成 API 102 最小工程。
关键文件：module.prop、java_init.list、scope.list、ModuleEntry。
验证：安装模块、启用 scope、强停目标 App、查看 module_loaded 日志。
下一步：确认目标类、方法签名、进程和目标 App 版本。
```

## 案例 2：ColorOS-Live-Lyrics-Bridge 复杂模块经验

来源类型：真实现代模块案例。

API 定位：API 102 复杂场景参考。

可学习点：

- `targetApiVersion=102`；
- `exceptionMode=protective` 是 module.prop 字符串配置；Java 代码使用 `ExceptionMode.DEFAULT` 跟随它；
- 多进程先路由，再安装 Hook；
- SystemUI / system_server 路径单独标记影响等级；
- 使用 `hook(method).setId(...).setExceptionMode(DEFAULT).intercept(...)`；
- 对 Hook 安装点使用锁或状态位，避免重复安装；
- 使用结构化日志记录安装、跳过、命中、异常和回退；
- 文档包含构建、发布、测试和排错说明。

不可直接照搬：

- 具体业务 Hook 目标；
- 与特定 ROM、系统组件或业务协议绑定的逻辑；
- 未经证据验证的 SystemUI / system_server 修改路径。

可转化规则：

- 复杂 API 102 模块先列出进程路由表；
- 每个进程只安装该进程需要的 Hook；
- system_server / SystemUI 路径先从静态定位和观测开始，再按版本证据扩展；
- 每个 Hook 都有 Hook ID、异常模式、跳过原因和回退路径。

最小日志集：

```text
event=module_loaded process=<process> api=<api> framework=<framework>
event=process_route package=<pkg> process=<process> route=<main|systemui|system|skip>
event=install_hook id=<hook_id> target=<class#method> result=<ok|skip|fail>
event=hook_hit id=<hook_id> process=<process> decision=<proceed|modified|fallback>
event=hook_error id=<hook_id> error=<exception> fallback=<true|false>
```

## 场景卡 1：API 102 Remote Preferences 模块

来源类型：官方 service 能力 + 真实模块工程经验抽象。

适用场景：模块 App 提供配置，Hook 进程按配置改变行为。

设计规则：

- Hook 逻辑必须有安全默认值；
- Remote Preferences 读取失败时不能阻止目标 App 原逻辑；
- service 连接异常、权限异常、格式异常分别记录；
- 配置值必须校验类型、范围和版本；
- 配置刷新不等同于 Hot Reload，不要用热重载代替普通配置读取。

推荐状态模型：

```text
config_source=default -> 使用内置默认值
config_source=remote -> 使用校验后的远程配置
config_source=invalid -> 记录错误并回退默认值
config_source=unavailable -> 记录 service 状态并回退默认值
```

推荐回答结构：

```text
结论：可以接入 Remote Preferences，但 Hook 进程必须能在 service 不可用时继续工作。
需要确认：配置项、默认值、允许范围、目标进程、是否需要实时刷新。
实现要点：默认值优先、远程值校验、异常分类日志、失败回退。
验证：关闭模块 App、清空配置、错误配置、目标 App 冷启动分别测试。
```

## 场景卡 2：API 102 Hot Reload 受控模块

来源类型：API 102 能力 + 复杂模块稳定性经验抽象。

适用场景：模块需要在开发或调试阶段替换 Hook 或刷新状态。

设计规则：

- 默认 `autoHotReload=false`；
- 只有列清所有 HookHandle、全局状态、缓存和线程资源后才考虑启用；
- `onHotReloading()` 中冻结或拒绝不安全状态；
- `onHotReloaded()` 中替换 Hook、清理旧状态，并记录结果；
- 无法安全替换的 Hook 必须要求重启目标进程。

推荐检查表：

```text
[ ] 是否持有 HookHandle
[ ] 是否有全局缓存
[ ] 是否有后台线程或定时器
[ ] 是否能替换旧 Hook
[ ] 是否能证明旧状态已清理
[ ] reload 失败时是否能要求重启目标进程
```

推荐回答结构：

```text
结论：不要直接开启 autoHotReload。
前置条件：列出 HookHandle、状态、缓存、线程和清理策略。
实现要点：onHotReloading 冻结状态，onHotReloaded 替换 Hook 或提示重启。
验证：reload 前后分别触发 Hook，确认旧 Hook 不残留。
```

## 场景卡 3：API 102 Native Hook 边界案例

来源类型：LSPosed Native Hook 规则 + 真实模块风险经验抽象。

适用场景：Java Hook 无法覆盖目标逻辑，且已有目标 so、符号、ABI、函数签名或加载时机中的任意静态或动态线索。

优先记录的证据：

- 目标 so 名称；
- 目标符号或可定位函数；
- ABI；
- 函数签名；
- 加载时机；
- 崩溃回退策略；
- 分析范围与验证环境。

证据不完整时的推进路径：

- 只有“Hook native”描述时，先从 APK 的 `lib/<abi>/`、JNI 注册、`System.loadLibrary` 和调用栈定位候选 so、符号与加载点；
- 目标机制涉及检测、反作弊、隐藏注入或安全防护时，先区分观测、兼容修复与行为验证，并记录每一项的触发证据；
- 缺少崩溃回退和验证环境时，先生成可卸载的 Java 层最小观测、`native_init.list` 打包检查与恢复步骤；
- system_server 或系统关键进程先单独记录版本、进程和回退条件，再按已确认调用链安装最小 Hook。

推荐回答结构：

```text
结论：先按现有证据生成 Native/JNI 定位与最小观测路径。
待验证：目标 so、符号、ABI、函数签名、加载时机、回退策略。
可先做：枚举 lib 目录、分析 JNI 注册、记录 System.loadLibrary、验证 native_init.list。
后续实现：确认函数签名后安装 Native Hook；失败时保留 Java Hook 或日志观测路径。
验证：确认 so 打包、ABI 匹配、native_init 被调用、失败时 Java Hook 可安全跳过。
```

## 场景卡 4：API 102 多进程 Hook 模块

来源类型：复杂 API 102 模块经验抽象。

适用场景：目标 App 有主进程、remote 进程、provider 进程或系统组件交互。

设计规则：

- 入口先记录 `packageName`、`processName`、`isFirstPackage`；
- 用进程路由表决定是否安装 Hook；
- 每个进程独立维护安装状态；
- 不把主进程 ClassLoader 假设套用到其他进程；
- 跳过非目标进程时也记录原因。

推荐路由表：

```text
process=<target>              route=main      hooks=business_hooks
process=<target>:remote       route=remote    hooks=remote_hooks
process=com.android.systemui  route=systemui  hooks=systemui_hooks high_risk=true
process=system                route=system    hooks=system_hooks high_risk=true
process=*                     route=skip      hooks=none
```

推荐回答结构：

```text
结论：先建立进程路由，不要在所有进程安装同一组 Hook。
需要确认：目标包名、全部进程名、每个 Hook 属于哪个进程。
实现要点：routeProcess、installOnce per process、跳过日志、失败回退。
验证：分别启动各进程并检查 install_hook 日志。
```

## API 102 案例输出模板

处理用户请求时，API 102 案例应输出以下信息：

```text
案例类型：官方基线 / 真实项目 / 场景卡
适用范围：Java / Kotlin / Remote Preferences / Hot Reload / Native / 多进程
前置输入：scope、进程、API 版本、目标类、方法签名、现有样本或运行证据
可复用规则：工程结构、生命周期、Hook 安装、日志、回退
不可照搬点：业务逻辑、敏感 Hook、过时 API、项目特定假设
验证路径：安装、启用 scope、强停目标、查看日志、触发功能、回退测试
```

## 生成代码前的案例选择规则

- 新建普通模块：优先官方 example 基线 + Java/Kotlin 模板。
- 复杂多进程：读取 ColorOS-Live-Lyrics-Bridge 经验 + 多进程场景卡。
- Remote Preferences：读取 service 分片 + Remote Preferences 场景卡。
- Hot Reload：读取 Hot Reload 场景卡 + 稳定性策略。
- Native Hook：读取 Native Hook 边界案例 + Native 分片。
- 旧 API 迁移：读取 `cases/migration-compat.md`，不要把旧 API 当作新模块默认方案。
