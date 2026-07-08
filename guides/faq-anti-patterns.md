# FAQ 与常见反模式速查

这个文件把分散在安全边界、案例、迁移和排错分片里的高频问题集中到一处。遇到不确定请求时，先用这里判断方向，再按需读取 `knowledge/` 分片。

## FAQ

### 1. 模块为什么安装后没有效果？

优先检查四件事：

- `module.prop` 是否被打包到 `META-INF/xposed/module.prop`；
- `java_init.list` 是否指向真实入口类；
- `scope.list` 是否包含目标包名；
- 是否重启目标 App、强停目标 App 或重启系统服务。

如果四项都正确，再检查 ClassLoader、进程名、方法签名和 Hook 时机。

### 2. API 102 模块为什么不能把 `libxposed:api` 写成 `implementation`？

`io.github.libxposed:api:102.0.0` 应使用 `compileOnly`。运行时 API 由框架提供，打进 APK 会增加冲突和误判风险。

### 3. 什么时候使用 `libxposed/helper`？

只在复杂查找、混淆定位或需要现代辅助能力时使用。不要默认加入 helper，也不要固定旧版本号；生成前必须以 Maven/Gradle 实际可解析版本为准。

### 4. 什么时候开启 `autoHotReload=true`？

只有模块已经实现清晰的状态清理、Hook 替换和 reload 回调时才开启。默认保持 `false` 更稳妥。

### 5. 旧 `XposedHelpers` 是否还能用？

只用于迁移或兼容场景。现代 API 102 模块应优先使用 `XposedModule` 生命周期、Hooker、HookHandle、Invoker 等现代模型。

### 6. Native Hook 是否适合默认生成？

不适合。Native Hook 需要明确目标 so、符号、ABI、加载时机和回退策略。只有用户明确需要 Native Hook 时才展开。

## 常见反模式

| 反模式 | 风险 | 推荐做法 |
|---|---|---|
| scope 写得过大 | 增加误触发、性能和合规风险 | 只声明目标包名，必要时再扩展 |
| 入口类堆砌 | 生命周期混乱，日志难追踪 | 保留少量入口类，内部按策略分发 |
| 不判断包名和进程 | Hook 到错误进程 | 入口处先判断 package/process |
| 盲 Hook 私有混淆类 | 版本变动后失效 | 优先稳定公开路径，必要时做降级 |
| 方法签名靠猜 | `NoSuchMethod` 或误 Hook | 从源码、反编译或日志确认签名 |
| 缺少异常日志 | 出错后无法定位 | 捕获异常并记录 event/package/process/reason |
| 把 API 依赖打进 APK | 与框架运行时冲突 | `libxposed:api` 使用 `compileOnly` |
| 默认开启 Hot Reload | 旧 Hook 和状态残留 | 默认关闭，确认 reload 安全后再启用 |
| 旧 `XSharedPreferences` 直迁 | 新系统不可用或不稳定 | 使用 `libxposed/service` 与 Remote Preferences |
| Native Hook 不校验 ABI | 加载失败或崩溃 | 检查 ABI、so 名称、符号和加载日志 |

## 边界请求改写

| 用户原始方向 | 处理方式 | 可改写方向 |
|---|---|---|
| 绕过检测、反作弊、风控 | 拒绝 | 模块稳定性排错、日志诊断、防护分析 |
| 窃取隐私、凭据、聊天记录 | 拒绝 | 授权测试环境下的数据访问边界说明 |
| 隐蔽注入、隐藏模块 | 拒绝 | 合法模块可观测性和审计设计 |
| 未授权修改第三方 App | 拒绝 | 自有 App、测试 App 或授权环境兼容适配 |
| system_server 高风险 Hook | 先确认合法目的和回退方案 | 最小 Hook、日志、失败保护和验证流程 |

## 快速判断规则

- 目标不合法时，先拒绝危险目标，再给合法替代方向；
- 目标合法但信息不足时，先索要包名、scope、日志、版本和签名；
- 目标涉及复杂生命周期时，优先读取 `knowledge/01-project-basics.md` 和 `knowledge/02-hook-api.md`；
- 目标涉及旧 API 时，先读取 `cases/migration-compat.md`；
- 目标涉及架构质量时，先读取 `cases/real-project-patterns.md`。