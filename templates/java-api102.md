# API 102 Java 模块模板

## 适用场景

用于新建默认现代 API 102 LSPosed 模块。适合普通 App Hook、配置验证、最小模块骨架。

## ModuleEntry.java

```java
package com.example.module;

import androidx.annotation.NonNull;
import io.github.libxposed.api.XposedInterface;
import io.github.libxposed.api.XposedModule;
import io.github.libxposed.api.XposedModuleInterface;

public final class ModuleEntry extends XposedModule {
    private static final String TAG = "ExampleModule";
    private static final String TARGET_PACKAGE = "com.example.target";
    private volatile boolean installed;

    @Override
    public void onModuleLoaded(@NonNull XposedModuleInterface.ModuleLoadedParam param) {
        log(TAG + ": event=module_loaded process=" + param.getProcessName()
                + " api=" + getApiVersion()
                + " framework=" + getFrameworkName()
                + " version=" + getFrameworkVersion());
    }

    @Override
    public void onPackageReady(@NonNull XposedModuleInterface.PackageReadyParam param) {
        if (!TARGET_PACKAGE.equals(param.getPackageName()) || !param.isFirstPackage()) {
            return;
        }
        installHooks(param.getClassLoader());
    }

    private synchronized void installHooks(ClassLoader classLoader) {
        if (installed) {
            log(TAG + ": event=install_skipped reason=already_installed");
            return;
        }
        try {
            Class<?> target = classLoader.loadClass("com.example.target.TargetClass");
            var method = target.getDeclaredMethod("targetMethod", String.class);
            method.setAccessible(true);

            hook(method)
                    .setId("target_method_hook")
                    .setExceptionMode(XposedInterface.ExceptionMode.PROTECTIVE)
                    .intercept(chain -> {
                        Object arg0 = chain.getArg(0);
                        if (!(arg0 instanceof String)) {
                            return chain.proceed();
                        }
                        return chain.proceed();
                    });

            installed = true;
            log(TAG + ": event=hook_registered method=TargetClass.targetMethod");
        } catch (ClassNotFoundException e) {
            log(TAG + ": event=class_not_found", e);
        } catch (NoSuchMethodException e) {
            log(TAG + ": event=method_not_found", e);
        } catch (Throwable t) {
            log(TAG + ": event=install_failed", t);
        }
    }
}
```

## 生成要求

生成真实项目时必须替换：

- `package com.example.module`；
- `TARGET_PACKAGE`；
- 目标类名；
- 目标方法名；
- 参数类型；
- Hook ID；
- 日志 TAG。

## 质量要求

- 必须判断包名；
- 必须判断 `isFirstPackage()` 或进程；
- 必须使用目标进程 `ClassLoader`；
- 必须设置异常保护；
- 条件不满足必须 `chain.proceed()`；
- 不要默认 Hook system_server；
- 不要把旧 `XposedHelpers` 作为默认写法。