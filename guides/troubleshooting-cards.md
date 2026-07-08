# 快速排错卡片

这个文件用于把高频故障压缩成一页式排查路径。先按对应卡片排查，再按需读取完整知识库排错章节和案例。

## 卡片 1：模块不生效

先确认：

- APK 已安装，模块已在 LSPosed 中启用；
- scope 包含目标 App；
- 修改 scope 后已强停目标 App 或重启；
- `META-INF/xposed/module.prop` 已打包；
- `META-INF/xposed/java_init.list` 已打包且入口类名正确；
- `minApiVersion` / `targetApiVersion` 与当前框架兼容。

需要用户提供：

- `module.prop`；
- `java_init.list`；
- `scope.list`；
- LSPosed 模块日志；
- 目标包名和进程名。

## 卡片 2：Hook 不触发

按顺序检查：

1. 包名判断是否过早返回；
2. 进程判断是否匹配目标进程；
3. 使用的 ClassLoader 是否来自 `onPackageLoaded()` 或正确生命周期；
4. 类名是否被混淆或版本变动；
5. 方法名、参数类型、返回值是否精确；
6. Hook 安装异常是否被日志记录；
7. Hook 点是否在目标调用路径上。

建议日志至少包含：

```text
event=install_hook package=<pkg> process=<process> class=<class> method=<method> result=<ok|fail> reason=<reason>
```

## 卡片 3：目标 App 崩溃

优先判断是否由模块引起：

- 禁用模块后崩溃是否消失；
- 只保留最小 scope 后是否复现；
- 只开启单个 Hook 点是否复现；
- Hook 回调中是否修改了 null、类型或状态假设；
- 是否吞掉异常导致后续状态不一致。

处理策略：

- 使用 `ExceptionMode.PROTECTIVE`；
- Hook 回调内部只做最小逻辑；
- 复杂逻辑移到可测试函数；
- 出错时记录并回退原始返回值或跳过修改。

## 卡片 4：Remote Preferences / Service 失败

先检查：

- 是否确实需要模块 App 与 Hook 进程通信；
- 是否使用 `io.github.libxposed:service:102.0.0`；
- service API 是否只在支持的框架版本下调用；
- 是否捕获 `ServiceException`；
- 是否处理 `UnsupportedOperationException`；
- 远程文件或配置是否有空返回、权限和生命周期处理。

需要用户提供：

- service 初始化代码；
- 调用点生命周期；
- 失败堆栈；
- 配置键名和默认值策略。

## 卡片 5：Hot Reload 失败

先确认：

- `autoHotReload` 是否真的需要开启；
- 模块是否实现 `onHotReloading()` / `onHotReloaded()`；
- 是否清理旧状态和旧 Hook；
- 是否使用 `HookHandle.replaceHook(...)` 替换仍需保留的 Hook；
- 回调是否可能在 Binder 线程执行；
- service 返回状态是 `SUCCESS`、`IN_PROGRESS`、`PROCESS_DIED` 还是 `FAILED`。

默认建议：

- 新模块保持 `autoHotReload=false`；
- 只有验证 reload 安全后再改为 `true`；
- 失败时回到重启目标进程的稳定路径。

## 卡片 6：Native Hook 失败

先检查：

- `native_init.list` 是否被打包；
- so 是否被打包到正确 ABI 目录；
- 目标设备 ABI 是否匹配；
- `System.loadLibrary()` 是否执行；
- `native_init` 导出符号是否存在；
- `NativeAPIEntries.version` 是否正确处理；
- 目标 so 和目标符号是否已加载；
- 失败时是否有 Java 层和 native 层日志。

不要在信息不足时生成 Native Hook。必须先确认目标 so、符号、函数签名、ABI 和合法目的。

## 卡片 7：旧 API 迁移失败

常见原因：

- 仍按 `IXposedHookLoadPackage` 思路组织生命周期；
- 继续依赖旧 `XSharedPreferences`；
- 把 `assets/xposed_init` 当作现代入口；
- 没有迁移到 `META-INF/xposed/java_init.list`；
- 没有从旧 helper 迁移到现代 Hooker / HookHandle 模型。

建议读取：

- `cases/migration-compat.md`；
- 完整知识库旧 API 兼容与迁移章节；
- Java/Kotlin API 102 模板。

## 输出格式建议

排错回答优先按以下结构输出：

```text
最可能原因
证据
下一步检查
需要用户补充的信息
修复建议
验证步骤
```