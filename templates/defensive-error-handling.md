# 防御性代码与错误码模板

本模板用于生成或审查 API 102 LSPosed 模块代码时统一错误码、日志字段、Guard、异常保护和回退策略。它不是完整业务代码；生成 Java/Kotlin 模块时，应先读取对应语言模板，再按本文件补充最小必要防御逻辑。

## 使用时机

优先读取本文件的场景：

- 用户要求生成 Hook 代码且包含配置、反射、ClassLoader、Remote Preferences、Hot Reload 或 Native 分支；
- 用户要求排查模块不生效、Hook 不触发、崩溃或配置读取失败；
- 用户要求代码质量审查、稳定性审查或发布前检查；
- 现有代码缺少结构化日志、错误码、失败回退或输入校验。

不要因为读取本模板就扩大功能范围。普通最小 Hook 只需要最少的 Guard、日志和回退。

## 错误码命名

格式：

```text
LSM-<DOMAIN>-<NNN>
```

常用 domain：

| Domain | 含义 |
|---|---|
| LOAD | 模块加载、入口和元数据 |
| SCOPE | scope、包名、进程路由 |
| HOOK | Hook 安装、Hook 命中、Hook 回调 |
| CL | ClassLoader、类查找、动态加载 |
| SIG | 方法、构造器、字段签名 |
| CFG | 本地配置、Remote Preferences、Remote Files |
| HR | Hot Reload 状态迁移 |
| NATIVE | native_init、so、ABI、符号 |
| API | libxposed API 版本、能力差异 |
| BOUNDARY | SystemUI、system_server、高风险边界 |
| VALIDATE | 构建、打包、运行验证 |

## 推荐错误码表

| 错误码 | 级别 | 场景 | 默认处理 |
|---|---|---|---|
| LSM-LOAD-001 | info | `onModuleLoaded` 执行 | 记录框架、API、进程 |
| LSM-LOAD-002 | warn | `module.prop`、入口类或 API 版本不匹配 | 停止生成 Hook，先修配置 |
| LSM-SCOPE-001 | info | 包名或进程不匹配 | 跳过，不报错 |
| LSM-SCOPE-002 | warn | scope 过大或包含系统进程 | 要求用户确认最小 scope |
| LSM-HOOK-001 | info | Hook 注册成功 | 记录 hook_id、类、方法 |
| LSM-HOOK-002 | warn | 重复安装 Hook | 跳过重复安装 |
| LSM-HOOK-003 | error | Hook 安装失败 | 禁用该 Hook 点，保留模块可加载 |
| LSM-HOOK-004 | warn | Hook 回调参数不符合预期 | `chain.proceed()` 回退 |
| LSM-CL-001 | warn | 目标类不存在 | 记录类名、ClassLoader，跳过 |
| LSM-CL-002 | warn | 动态加载时机过早 | 有边界地重试或等待稳定生命周期 |
| LSM-SIG-001 | warn | 方法签名不匹配 | 记录目标签名，跳过 |
| LSM-CFG-001 | warn | 配置不可用 | 使用默认值 |
| LSM-CFG-002 | warn | 配置值类型或范围非法 | 丢弃远程值，使用默认值 |
| LSM-HR-001 | warn | Hot Reload 状态不完整 | 保持 `autoHotReload=false` 或要求重启目标进程 |
| LSM-NATIVE-001 | warn | so、ABI 或符号缺失 | 禁用 native 分支 |
| LSM-API-001 | warn | API 版本低于模板要求 | 降级到兼容路径或停止生成 |
| LSM-BOUNDARY-001 | error | system_server、SystemUI 或 native 高风险条件不足 | 暂缓或拒绝实现 |
| LSM-VALIDATE-001 | error | APK 缺少 `META-INF/xposed` 必要文件 | 修复打包配置后再测试 |

## 日志字段

所有防御性日志优先使用同一组字段，便于排错和检索：

```text
code=<LSM-...>
event=<module_loaded|route_skip|install_hook|hook_hit|fallback|validate>
package=<target package>
process=<target process>
stage=<lifecycle or check stage>
target=<class/method/config/native symbol>
result=<ok|skip|fail>
reason=<short reason>
recover=<proceed|default|disable_hook|restart_required|ask_user>
```

最低要求：

- 成功日志必须包含 `event`、`package`、`process`、`result`；
- 失败日志必须包含 `code`、`reason`、`recover`；
- Hook 安装日志必须包含类名、方法名或 hook_id；
- 不要吞掉异常后没有日志；
- 不要在日志中输出隐私数据、凭据、token、完整用户内容。

## Java Guard 片段

### 包名和进程路由

```java
private static final String TARGET_PACKAGE = "com.example.target";
private static final String TARGET_PROCESS = "com.example.target";

private boolean shouldInstall(String packageName, String processName, boolean firstPackage) {
    if (!TARGET_PACKAGE.equals(packageName)) {
        log(Log.INFO, TAG, "code=LSM-SCOPE-001 event=route_skip result=skip reason=package_mismatch package=" + packageName + " process=" + processName + " recover=none");
        return false;
    }
    if (!TARGET_PROCESS.equals(processName) || !firstPackage) {
        log(Log.INFO, TAG, "code=LSM-SCOPE-001 event=route_skip result=skip reason=process_mismatch package=" + packageName + " process=" + processName + " recover=none");
        return false;
    }
    return true;
}
```

### Hook 安装防御

```java
private synchronized void installTargetHook(ClassLoader classLoader, String packageName, String processName) {
    if (installed) {
        log(Log.INFO, TAG, "code=LSM-HOOK-002 event=install_hook result=skip reason=already_installed package=" + packageName + " process=" + processName + " recover=none");
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
                        log(Log.WARN, TAG, "code=LSM-HOOK-004 event=hook_hit result=skip reason=arg_type_mismatch recover=proceed");
                        return chain.proceed();
                    }
                    return chain.proceed();
                });

        installed = true;
        log(Log.INFO, TAG, "code=LSM-HOOK-001 event=install_hook result=ok target=TargetClass.targetMethod package=" + packageName + " process=" + processName + " recover=none");
    } catch (ClassNotFoundException e) {
        log(Log.WARN, TAG, "code=LSM-CL-001 event=install_hook result=fail target=com.example.target.TargetClass reason=class_not_found recover=disable_hook", e);
    } catch (NoSuchMethodException e) {
        log(Log.WARN, TAG, "code=LSM-SIG-001 event=install_hook result=fail target=TargetClass.targetMethod reason=method_not_found recover=disable_hook", e);
    } catch (Throwable t) {
        log(Log.ERROR, TAG, "code=LSM-HOOK-003 event=install_hook result=fail reason=unexpected_error recover=disable_hook", t);
    }
}
```

### 配置读取防御

```java
private String readModeOrDefault() {
    try {
        String value = readRemoteString("mode", "default");
        if (!"default".equals(value) && !"compact".equals(value)) {
            log(Log.WARN, TAG, "code=LSM-CFG-002 event=read_config result=skip reason=invalid_mode recover=default");
            return "default";
        }
        return value;
    } catch (Throwable t) {
        log(Log.WARN, TAG, "code=LSM-CFG-001 event=read_config result=fail reason=remote_unavailable recover=default", t);
        return "default";
    }
}
```

`readRemoteString` 只是占位函数。生成真实代码时必须替换为项目实际 Remote Preferences 或本地配置读取 API。

## Kotlin Guard 片段

```kotlin
private fun shouldInstall(packageName: String, processName: String, firstPackage: Boolean): Boolean {
    if (packageName != TARGET_PACKAGE) {
        log(Log.INFO, TAG, "code=LSM-SCOPE-001 event=route_skip result=skip reason=package_mismatch package=$packageName process=$processName recover=none")
        return false
    }
    if (processName != TARGET_PROCESS || !firstPackage) {
        log(Log.INFO, TAG, "code=LSM-SCOPE-001 event=route_skip result=skip reason=process_mismatch package=$packageName process=$processName recover=none")
        return false
    }
    return true
}

private fun installTargetHook(classLoader: ClassLoader, packageName: String, processName: String) {
    if (!installed.compareAndSet(false, true)) {
        log(Log.INFO, TAG, "code=LSM-HOOK-002 event=install_hook result=skip reason=already_installed package=$packageName process=$processName recover=none")
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
                val value = chain.getArg(0) as? String
                    ?: return@intercept chain.proceed()
                chain.proceed(arrayOf(value))
            }

        log(Log.INFO, TAG, "code=LSM-HOOK-001 event=install_hook result=ok target=TargetClass.targetMethod package=$packageName process=$processName recover=none")
    } catch (t: ClassNotFoundException) {
        installed.set(false)
        log(Log.WARN, TAG, "code=LSM-CL-001 event=install_hook result=fail target=com.example.target.TargetClass reason=class_not_found recover=disable_hook", t)
    } catch (t: NoSuchMethodException) {
        installed.set(false)
        log(Log.WARN, TAG, "code=LSM-SIG-001 event=install_hook result=fail target=TargetClass.targetMethod reason=method_not_found recover=disable_hook", t)
    } catch (t: Throwable) {
        installed.set(false)
        log(Log.ERROR, TAG, "code=LSM-HOOK-003 event=install_hook result=fail reason=unexpected_error recover=disable_hook", t)
    }
}
```

## 生成代码前检查

在输出防御性代码前，先确认：

```text
目标包名
目标进程
目标类名
目标方法签名
目标 App / Android / LSPosed / API 版本
scope 是否最小
失败时回退方式
是否需要 Remote Preferences / Hot Reload / Native
```

缺少核心信息时，不直接生成最终 Hook。先输出缺失项和最小验证步骤。

## 回答模板

```text
结论：可以生成 / 信息不足 / 暂缓高风险能力
使用错误码：列出本次会用到的 LSM-... 代码
防御策略：包名/进程 Guard、签名校验、ExceptionMode.PROTECTIVE、失败回退
代码位置：入口类、Hook Strategy、ConfigStore、Logger
验证：APK 内容、scope、module_loaded、install_hook、hook_hit、fallback 日志
风险：哪些条件下跳过 Hook，哪些条件下需要用户补充信息
```

## 与其他文件配合

- Java 代码生成：先读 `templates/java-api102.md`，再读本文件。
- Kotlin 代码生成：先读 `templates/kotlin-api102.md`，再读本文件。
- 稳定性策略：读 `guides/stability-strategy.md`。
- 自动化验证或发布前检查：读 `guides/validation-checklist.md`。
- 特殊高风险场景：先读 `guides/special-boundaries.md`。
