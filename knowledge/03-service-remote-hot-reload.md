# 03 - Service、Remote 配置与 Hot Reload

覆盖原知识库第 22-27 章：Remote Preferences、Remote Files、`libxposed/service`、service 注册、scope 请求、运行中目标、Hot Reload。

## Remote Preferences

Remote Preferences 是现代 API 推荐的模块 App 与 Hook 进程共享小配置方式。

Hook 进程读取：

```java
SharedPreferences prefs = getRemotePreferences("default");
boolean enabled = prefs.getBoolean("enabled", false);
```

Hook 进程监听：

```java
prefs.registerOnSharedPreferenceChangeListener((sharedPreferences, key) -> {
    boolean enabled = sharedPreferences.getBoolean("enabled", false);
});
```

规则：Hook 进程通常只读；数据存储在 LSPosed 数据库；listener 需要强引用避免被 GC；普通配置变化应使用 Remote Preferences，不要用 Hot Reload 同步。

模块 App 侧通过 `libxposed/service` 写入：

```kotlin
val prefs = service.getRemotePreferences("default")
prefs.edit().putBoolean("enabled", true).apply()
```

## Remote Files

Remote Files 用于共享较大内容，例如规则文件、JSON 配置或 blob。

Hook 进程读取：

```java
try (ParcelFileDescriptor pfd = openRemoteFile("config.json")) {
    // read pfd.getFileDescriptor()
}
```

模块 App 写入：

```kotlin
service.openRemoteFile("config.json").use { pfd ->
    FileWriter(pfd.fileDescriptor).use {
        it.write("{}")
    }
}
```

规则：文件名不能包含路径分隔符，不能是 `.` 或 `..`；Hook 进程中只读；框架返回空文件描述符时应视为 service 异常；`deleteRemoteFile(name)` 返回 `false` 表示文件不存在，不要因此崩溃；所有 App 侧 remote file 调用都要处理 service 死亡或远程异常。

## libxposed/service 用途

`libxposed/service` 用于模块 App 与 LSPosed 框架通信，可用于：

- 查询 API 版本、框架名称和框架版本；
- 查询框架能力；
- 获取当前 scope；
- 动态请求添加或删除 scope；
- 获取和删除 Remote Preferences；
- 打开和删除 Remote Files；
- 查询运行中的 Hook 目标；
- 触发 Hot Reload。

边界：service 只在模块 App 侧使用，不要在 Hook 进程里依赖它完成高频逻辑；App 启动时 service 可能尚未绑定；service 死亡或 Binder 调用失败会表现为 `XposedService.ServiceException`；remote preferences、remote files 依赖框架 remote capability；scope 请求和 Hot Reload 都是异步路径，不要写成同步成功假设。

## service 注册

在模块 App 的 `Application` 中注册：

```kotlin
class App : Application(), XposedServiceHelper.OnServiceListener {
    companion object {
        @Volatile
        var service: XposedService? = null
            private set
    }

    override fun onCreate() {
        super.onCreate()
        XposedServiceHelper.registerListener(this)
    }

    override fun onServiceBind(service: XposedService) {
        App.service = service
    }

    override fun onServiceDied(service: XposedService) {
        App.service = null
    }
}
```

注意：`registerListener()` 应只调用一次；`onServiceBind()` 可能多次调用，因为可能存在多个 Xposed framework；复杂 UI 应展示 `frameworkName`、`frameworkVersion` 等信息供用户判断；必须处理 `onServiceDied()`；UI 层应监听 service 状态。

## 框架能力与 scope

框架信息：

```kotlin
service.apiVersion
service.frameworkName
service.frameworkVersion
service.frameworkVersionCode
service.frameworkProperties
```

能力判断：

```kotlin
val props = service.frameworkProperties
val supportSystem = props and XposedService.PROP_CAP_SYSTEM != 0L
val supportRemote = props and XposedService.PROP_CAP_REMOTE != 0L
val apiProtection = props and XposedService.PROP_RT_API_PROTECTION != 0L
```

请求 scope：

```kotlin
service.requestScope(
    listOf("com.example.target"),
    object : XposedService.OnScopeEventListener {
        override fun onScopeRequestApproved(approved: List<String>) {
        }

        override fun onScopeRequestFailed(message: String) {
        }
    }
)
```

规则：scope 请求需要用户批准；不应静默强制添加；失败时提示用户手动启用。

## 运行中目标

API 102 支持查询运行中目标：

```kotlin
if (service.apiVersion >= 102) {
    val targets = service.runningTargets
}
```

规则：`getRunningTargets()` 只在 service API >= 102 可用；返回对象可传给 `hotReloadModule(...)`；`pid`、`uid`、`processName`、`loadedVersionCode` 只用于显示和诊断，不要当作稳定目标身份。

目标状态包括：

- `UP_TO_DATE`；
- `STALE`；
- `RELOADING`；
- `FAILED`。

`STALE` 通常表示目标仍运行旧模块代码，可能适合热重载；`RELOADING` 表示已有热重载进行中；`FAILED` 表示上次热重载被旧模块拒绝或过程中抛异常。

## Hot Reload 基础

Hot Reload 是 API 102 的高级能力，用于模块 App 更新后，为已经运行的目标进程加载新的模块 generation。

触发来源：

- 模块 App 通过 `libxposed/service` 调用 `hotReloadModule(...)`；
- 模块 App 更新时，如果 `module.prop` 设置 `autoHotReload=true`，框架可尝试自动热重载；
- 自动热重载仍必须经过旧代码的 `onHotReloading()`，只有返回 `true` 才会继续。

限制：

- 只支持恰好一个 Java 入口类；
- 不支持零入口或多入口模块；
- 框架可能返回 `UNSUPPORTED`；
- 不保证 native library 立即卸载；
- 不自动重放生命周期；
- 不是配置同步机制；
- 同一个目标进程不会并发执行多个热重载请求。

## onHotReloading

旧代码中调用：

```java
@Override
public boolean onHotReloading(@NonNull HotReloadingParam param) {
    param.setSavedInstanceState("state");
    return true;
}
```

规则：默认返回 `false`；返回 `true` 表示允许热重载；返回 `false` 表示拒绝；返回前必须清理模块持有资源。冻结旧代码后，旧代码继续注册 Hook 会失败；已经开始执行的 Hook 调用继续使用其开始时的 chain 快照。

必须清理：Java 线程、native 线程、外部回调、native hooks、JNI global references、系统或 App 类中保存的模块对象引用。

## onHotReloaded

新代码中调用：

```java
@Override
public void onHotReloaded(@NonNull HotReloadedParam param) {
    Object state = param.getSavedInstanceState();
    for (HookHandle handle : param.getOldHookHandles()) {
        handle.unhook();
    }
}
```

规则：不会自动重新调用 `onModuleLoaded()`、`onPackageLoaded()` 或 `onPackageReady()`；需要在此处替换旧 Hook、移除不应保留的 Hook，或重新安装必要 Hook；可优先用 `HookHandle.replaceHook(...)` 原子替换仍需保留的 Hook；默认实现会 unhook 所有旧 Hook。

## service 触发 Hot Reload

```kotlin
if (service.apiVersion >= 102) {
    val targets = service.runningTargets
    for (target in targets) {
        service.hotReloadModule(target, null) { hookedTarget, result ->
            // handle result
        }
    }
}
```

规则：`target` 必须来自 `service.runningTargets`；结果通过 callback 异步返回；callback 可能运行在 Binder 线程，更新 UI 前必须切回主线程；调用前检查 service API >= 102，并处理 `UnsupportedOperationException`、`XposedService.ServiceException` 和 `SecurityException`；`data` 只能包含 classloader-neutral 的值，不要放模块自定义 `Parcelable` 或 `Serializable` 对象。

结果状态：`SUCCEEDED`、`FAILED`、`UNSUPPORTED`、`IN_PROGRESS`、`PROCESS_DIED`。

排错提示：`FAILED` 且 message 为 null 通常表示旧模块返回 `false`；非 null message 表示框架提供的错误诊断；`IN_PROGRESS` 表示同一目标已有热重载进行中；`PROCESS_DIED` 表示目标进程在热重载过程中退出。