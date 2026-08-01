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
| LSM-BOUNDARY-001 | error | system_server、SystemUI 或 native 路径缺少版本/回退信息 | 先输出静态定位、最小观测和恢复步骤 |
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
                .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
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
            .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
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
防御策略：包名/进程 Guard、签名校验、ExceptionMode.DEFAULT（由 module.prop 配置 protective/passthrough）、失败回退
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

## 快速诊断决策树

遇到模块问题时，按以下顺序快速定位：

```text
1. 模块是否加载？
   ├─ 否 → 检查 LSM-LOAD-001/002：module.prop、java_init.list、scope.list
   └─ 是 → 继续

2. Hook 是否安装？
   ├─ 否 → 检查 LSM-CL-001/LSM-SIG-001：类/方法是否存在
   └─ 是 → 继续

3. Hook 是否触发？
   ├─ 否 → 检查 LSM-SCOPE-001：包名/进程是否匹配
   └─ 是 → 继续

4. Hook 逻辑是否正确？
   ├─ 否 → 检查 LSM-HOOK-004：参数类型、返回值、异常
   └─ 是 → 检查业务逻辑

5. 配置是否生效？
   └─ 检查 LSM-CFG-001/002：Remote Preferences 可用性、值合法性
```

### 常见错误码速查

| 错误码 | 快速判断 | 立即操作 |
|---|---|---|
| LSM-LOAD-002 | module.prop 或入口类配置错误 | 检查 APK 中 META-INF/xposed/ 文件 |
| LSM-SCOPE-001 | 包名/进程不匹配 | 确认 scope.list 和目标 App 包名 |
| LSM-HOOK-003 | Hook 安装失败 | 查看异常栈，检查类/方法签名 |
| LSM-CL-001 | 类不存在 | 确认 ClassLoader 和 Hook 时机 |
| LSM-SIG-001 | 方法签名不匹配 | 反编译确认实际签名 |
| LSM-CFG-001 | 配置不可用 | 检查 Remote Preferences 是否初始化 |
| LSM-NATIVE-001 | Native Hook 失败 | 检查 ABI、so 和符号 |

### 一键验证脚本

生成模块后，使用此脚本快速验证：

```bash
#!/bin/bash
# verify-module.sh

APK="$1"
if [[ ! -f "$APK" ]]; then
    echo "用法: $0 <module.apk>"
    exit 1
fi

echo "=== 模块验证 ==="
echo ""

echo "1. 检查 module.prop..."
unzip -p "$APK" META-INF/xposed/module.prop && echo "✓ module.prop 存在" || echo "✗ module.prop 缺失"
echo ""

echo "2. 检查 java_init.list..."
unzip -p "$APK" META-INF/xposed/java_init.list && echo "✓ java_init.list 存在" || echo "✗ java_init.list 缺失"
echo ""

echo "3. 检查 scope.list..."
unzip -p "$APK" META-INF/xposed/scope.list && echo "✓ scope.list 存在" || echo "✗ scope.list 缺失"
echo ""

echo "4. 检查 classes.dex..."
unzip -l "$APK" | grep "classes.*\.dex" && echo "✓ DEX 文件存在" || echo "✗ DEX 文件缺失"
echo ""

echo "5. 检查 libxposed API（不应打包）..."
unzip -l "$APK" | grep "io/github/libxposed/api" && echo "✗ 警告：API 被打包了（应使用 compileOnly）" || echo "✓ API 未打包"
echo ""

echo "=== 验证完成 ==="
```

### 日志模板生成器

快速生成标准化日志代码：

```java
// Java 日志模板
private static void logInfo(String event, String packageName, String processName, String result) {
    String msg = String.format("event=%s package=%s process=%s result=%s", 
        event, packageName, processName, result);
    Log.i(TAG, msg);
}

private static void logError(String code, String event, String reason, String recover, Throwable t) {
    String msg = String.format("code=%s event=%s reason=%s recover=%s", 
        code, event, reason, recover);
    Log.e(TAG, msg, t);
}

// 使用示例
logInfo("install_hook", packageName, processName, "ok");
logError("LSM-HOOK-003", "install_hook", "unexpected_error", "disable_hook", t);
```

```kotlin
// Kotlin 日志模板
private fun logInfo(event: String, packageName: String, processName: String, result: String) {
    Log.i(TAG, "event=$event package=$packageName process=$processName result=$result")
}

private fun logError(code: String, event: String, reason: String, recover: String, t: Throwable? = null) {
    Log.e(TAG, "code=$code event=$event reason=$reason recover=$recover", t)
}

// 使用示例
logInfo("install_hook", packageName, processName, "ok")
logError("LSM-HOOK-003", "install_hook", "unexpected_error", "disable_hook", t)
```
