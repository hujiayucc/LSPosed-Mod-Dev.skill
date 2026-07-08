# 稳定性策略与降级路径

这个文件用于补充模板之外的稳定性规则，重点覆盖重试、超时、参数校验、状态保护和失败降级。生成代码时不要把这里当作固定代码模板，而应按场景选取最小必要策略。

## 总原则

- 默认失败可见：任何失败都要留下可诊断日志；
- 默认安全回退：Hook 失败时不应破坏目标 App 原有流程；
- 默认最小重试：只在明确生命周期变化或动态加载时重试；
- 默认不阻塞：不要在 Hook 回调或 Binder 回调里做长耗时操作；
- 默认参数校验：外部输入、远程配置和反射结果都必须校验后使用。

## 参数校验

生成或审查代码时，先确认以下输入：

| 输入 | 必须校验 |
|---|---|
| 目标包名 | 非空、等于当前 `packageName` |
| 目标进程 | 非空、符合预期进程或允许列表 |
| 类名 | 非空，失败时记录 ClassLoader 和类名 |
| 方法签名 | 参数类型、返回值和重载版本精确匹配 |
| Remote Preferences | 键名、默认值、类型和值域 |
| Remote Files | 文件名、空返回、读取异常和大小上限 |
| Hot Reload data | 只使用基础类型、数组或框架允许对象 |
| Native Hook | so 名称、符号、ABI、函数签名和版本 |

参数不完整时，不要直接生成最终 Hook。应先输出缺失信息和最小定位步骤。

## Hook 安装重试

只在以下情况考虑重试：

- 目标类由动态 dex、插件或分包延迟加载；
- 首次生命周期过早，后续有稳定初始化点；
- 目标进程刚启动，关键类尚未完成初始化。

不要这样做：

- 无限循环查找类；
- 在主线程频繁 sleep；
- Hook 回调里反复安装同一个 Hook；
- 吞掉 `ClassNotFoundException` 后不给日志。

推荐策略：

```text
first_try = onPackageLoaded()
if class_missing:
  log classloader + class name
  wait for stable loader point
  retry once or bounded times
if still_missing:
  disable this hook point and keep module alive
```

## 超时与耗时操作

LSPosed Hook 回调、Hot Reload 回调和 service 相关回调都应避免长耗时逻辑。

需要超时意识的场景：

- 远程配置读取；
- Remote Files 读取；
- 网络、数据库或磁盘操作；
- Binder 回调；
- Hot Reload 状态迁移；
- Native 初始化。

建议：

- Hook 回调里只读取已经准备好的轻量状态；
- 远程配置失败时用默认值；
- 文件读取失败时跳过增强逻辑；
- Binder 或 Hot Reload 回调中不要做重计算；
- 没有可靠超时 API 时，用失败回退替代等待。

## 状态保护

常见状态来源：

- `AtomicBoolean` 或安装状态；
- HookHandle 列表；
- Remote Preferences 缓存；
- Native 初始化状态；
- Hot Reload 旧模块状态；
- 目标 App 版本或进程状态。

建议：

- 用明确状态标记避免重复安装；
- 每个 Hook 点独立记录成功/失败；
- Hot Reload 前列出需要保留和需要清理的状态；
- Native 初始化失败时不要继续调用 native 逻辑；
- 配置变更失败时保留上一个有效配置或回退默认值。

## 降级路径

| 失败点 | 降级策略 |
|---|---|
| 类不存在 | 记录并跳过该 Hook 点 |
| 方法不存在 | 记录签名，跳过修改 |
| Hook 回调异常 | 保留原始返回值或跳过修改 |
| Remote Preferences 不可用 | 使用默认配置 |
| Remote Files 不可用 | 跳过依赖文件的增强逻辑 |
| service API 不支持 | 捕获异常并禁用相关能力 |
| Hot Reload 失败 | 回到重启目标进程路径 |
| Native 初始化失败 | 禁用 native 分支，保留 Java 分支 |

## 日志字段

稳定性相关日志至少包含：

```text
event=<事件>
package=<目标包名>
process=<目标进程>
stage=<生命周期或阶段>
target=<类/方法/文件/符号>
result=<ok|skip|fail>
reason=<失败原因>
recover=<降级方式>
```

## 生成代码时的最低要求

当用户要求生成复杂代码时，回答必须明确：

- 哪些输入已确认；
- 哪些输入仍缺失；
- 是否需要重试；
- 是否需要超时或失败回退；
- 参数校验放在哪里；
- 失败后如何保持目标 App 稳定；
- 如何通过日志验证稳定性。

## 不建议主动加入的机制

除非用户明确需要，不要默认加入：

- 后台常驻任务；
- 周期性轮询；
- 大范围 Hook 扫描；
- 全局状态热替换；
- Native Hook；
- 自动开启 Hot Reload；
- 跨进程复杂同步。

这些机制会显著增加排错难度，应先用最小模块验证核心 Hook。