# API 102 Kotlin 模块模板

## 适用场景

用于新建 Kotlin LSPosed 模块。适合需要清晰分层、Hook Strategy、Logger、Guard 的项目。

## ModuleEntry.kt

```kotlin
package com.example.module

import android.util.Log
import io.github.libxposed.api.XposedInterface
import io.github.libxposed.api.XposedModule
import io.github.libxposed.api.XposedModuleInterface
import java.util.concurrent.atomic.AtomicBoolean

class ModuleEntry : XposedModule() {
    private val installed = AtomicBoolean(false)
    @Volatile
    private var loadedProcess: String? = null

    override fun onModuleLoaded(param: XposedModuleInterface.ModuleLoadedParam) {
        loadedProcess = param.processName
        log(Log.INFO, TAG, "event=module_loaded process=${param.processName} api=${getApiVersion()} framework=${getFrameworkName()}")
    }

    override fun onPackageReady(param: XposedModuleInterface.PackageReadyParam) {
        if (param.packageName != TARGET_PACKAGE || loadedProcess != TARGET_PROCESS) return
        installHooks(param.classLoader)
    }

    private fun installHooks(classLoader: ClassLoader) {
        if (!installed.compareAndSet(false, true)) {
            log(Log.INFO, TAG, "event=install_skipped reason=already_installed")
            return
        }

        try {
            val target = classLoader.loadClass("com.example.target.TargetClass")
            val method = target.getDeclaredMethod("targetMethod", String::class.java)
            method.isAccessible = true

            hook(method)
                .setId("target_method_hook")
                .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
                .intercept { chain ->
                    val value = chain.getArg(0) as? String ?: return@intercept chain.proceed()
                    chain.proceed(arrayOf(value))
                }

            log(Log.INFO, TAG, "event=hook_registered method=TargetClass.targetMethod")
        } catch (t: Throwable) {
            installed.set(false)
            log(Log.ERROR, TAG, "event=install_failed", t)
        }
    }

    private companion object {
        const val TAG = "ExampleModule"
        const val TARGET_PACKAGE = "com.example.target"
        const val TARGET_PROCESS = "com.example.target"
    }
}
```

## 推荐分层

```text
ModuleEntry.kt
hook/
  TargetProcessEntrypoint.kt
  XxxHookStrategy.kt
guard/
  PackageGuard.kt
  RuntimeGuard.kt
state/
  ConfigStore.kt
util/
  Logger.kt
```

## 质量要求

- 入口类只负责生命周期和路由；
- Hook Strategy 负责具体 Hook；
- 使用 `AtomicBoolean` 或 key 防止重复安装；
- 条件不满足必须回退原逻辑；
- 默认使用 `ExceptionMode.DEFAULT`，由 `module.prop` 设置稳定异常模式；
- 日志必须包含 event、package、process、reason；
- 不要默认生成 system_server、SystemUI 或 native Hook。