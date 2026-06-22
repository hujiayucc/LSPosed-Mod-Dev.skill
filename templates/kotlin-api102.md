# API 102 Kotlin 模块模板

## 适用场景

用于新建 Kotlin LSPosed 模块。适合需要清晰分层、Hook Strategy、Logger、Guard 的项目。

## ModuleEntry.kt

```kotlin
package com.example.module

import io.github.libxposed.api.XposedInterface
import io.github.libxposed.api.XposedModule
import io.github.libxposed.api.XposedModuleInterface
import java.util.concurrent.atomic.AtomicBoolean

class ModuleEntry : XposedModule() {
    private val installed = AtomicBoolean(false)

    override fun onModuleLoaded(param: XposedModuleInterface.ModuleLoadedParam) {
        log("ExampleModule: event=module_loaded process=${param.processName} api=${getApiVersion()} framework=${getFrameworkName()}")
    }

    override fun onPackageReady(param: XposedModuleInterface.PackageReadyParam) {
        if (param.packageName != TARGET_PACKAGE || !param.isFirstPackage) return
        installHooks(param.classLoader)
    }

    private fun installHooks(classLoader: ClassLoader) {
        if (!installed.compareAndSet(false, true)) {
            log("ExampleModule: event=install_skipped reason=already_installed")
            return
        }

        try {
            val target = classLoader.loadClass("com.example.target.TargetClass")
            val method = target.getDeclaredMethod("targetMethod", String::class.java)
            method.isAccessible = true

            hook(method)
                .setId("target_method_hook")
                .setExceptionMode(XposedInterface.ExceptionMode.PROTECTIVE)
                .intercept { chain ->
                    val value = chain.getArg(0) as? String ?: return@intercept chain.proceed()
                    chain.proceed(arrayOf(value))
                }

            log("ExampleModule: event=hook_registered method=TargetClass.targetMethod")
        } catch (t: Throwable) {
            log("ExampleModule: event=install_failed", t)
        }
    }

    private companion object {
        const val TARGET_PACKAGE = "com.example.target"
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
- 日志必须包含 event、package、process、reason；
- 不要默认生成 system_server、SystemUI 或 native Hook。