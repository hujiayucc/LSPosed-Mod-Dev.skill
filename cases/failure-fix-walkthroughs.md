# 真实故障修复过程与修复前后对比

本文件用于补齐“典型错误代码 -> 排查证据 -> 修复后代码 -> 验证结果”的案例。它不替代 `guides/troubleshooting-cards.md` 和 `guides/validation-checklist.md`；遇到模块不生效、Hook 不触发、崩溃、配置失败或 Hot Reload 异常时，先按排错卡片定位层级，再用本文件选择相近案例。

所有案例默认前提：目标 App、设备、版本和测试行为均可复现；输入需提供样本来源、分析范围和脱敏日志；涉及数据、系统稳定性或高影响路径时，保留最小 scope、崩溃保护与回滚方案。

## 使用原则

- 先判断失败层级：配置、scope、生命周期、ClassLoader、方法签名、Hook 回调、配置读取、Hot Reload 或 Native。
- 修复前后对比只改最小必要点，不扩大 scope，不顺手换 Hook 目标。
- 每个修复都必须给出证据、错误码、日志变化和验证步骤。
- 用户只给出片段时，先说明还缺哪些文件或日志，不假设真实包名、进程名、类名或方法签名。
- 代码片段只作为模板，生成真实项目时必须替换包名、类名、方法签名和日志实现。

## 快速选择表

| 现象 | 优先案例 | 主要错误码 | 推荐读取 |
|---|---|---|---|
| 完全没有 `module_loaded` | 案例 1 | LSM-LOAD-002 / LSM-VALIDATE-001 | `guides/validation-checklist.md` |
| `module_loaded` 有，但目标进程没有日志 | 案例 2 | LSM-SCOPE-001 / LSM-SCOPE-003 | `guides/troubleshooting-cards.md` |
| `install_hook` 失败或类找不到 | 案例 3 | LSM-CL-001 / LSM-HOOK-002 | `knowledge/02-hook-api.md` |
| Hook 安装成功但不命中 | 案例 4 | LSM-SIG-001 / LSM-HOOK-003 | `knowledge/02-hook-api.md` |
| Hook 后目标 App 崩溃 | 案例 5 | LSM-HOOK-004 / LSM-HOOK-005 | `templates/defensive-error-handling.md` |
| Remote Preferences 配置异常 | 案例 6 | LSM-CFG-001 / LSM-CFG-002 | `knowledge/03-service-remote-hot-reload.md` |
| Hot Reload 后状态重复或旧 Hook 残留 | 案例 7 | LSM-HR-001 / LSM-HR-002 | `guides/stability-strategy.md` |
| Native 分支加载失败 | 案例 8 | LSM-NATIVE-001 / LSM-NATIVE-002 | `knowledge/04-native-migration-helper.md` |

## 案例 1：APK 缺少现代入口文件导致模块完全不加载

### 用户输入

```text
模块在 LSPosed 里启用了，scope 也勾了目标 App，但目标 App 启动后没有任何 module_loaded 日志。这是 APK 内容列表和 module.prop。
```

### 证据判断

- 日志链路缺少 `event=module_loaded`。
- APK 内容列表没有 `META-INF/xposed/java_init.list`。
- 项目仍可能使用旧入口思路，例如只配置了 `assets/xposed_init`。

### 修复前

```text
APK contents:
assets/xposed_init
META-INF/xposed/module.prop
META-INF/xposed/scope.list
```

```properties
# module.prop
id=demo
name=Demo Module
version=1.0
minApiVersion=100
targetApiVersion=102
```

### 问题

现代 API 102 模块入口必须通过 `META-INF/xposed/java_init.list` 指向入口类。没有入口文件时，框架无法加载模块入口，后续 Hook 代码不会执行。

### 修复后

```text
APK contents:
META-INF/xposed/module.prop
META-INF/xposed/java_init.list
META-INF/xposed/scope.list
```

```text
# META-INF/xposed/java_init.list
com.example.lsposed.demo.ModuleEntry
```

```properties
# META-INF/xposed/module.prop
id=demo
name=Demo Module
version=1.0
minApiVersion=102
targetApiVersion=102
```

### 回答模板

```text
结论：当前优先修工程配置，不排查 Hook 逻辑。
错误码：LSM-LOAD-002、LSM-VALIDATE-001。
证据：APK 内缺少 META-INF/xposed/java_init.list，运行日志也没有 module_loaded。
修复：补齐 java_init.list，并确认入口类继承 XposedModule。
验证：重新安装模块 -> 启用 scope -> 强停目标 App -> 查看 event=module_loaded result=ok。
```

## 案例 2：scope 或进程路由不匹配导致 Hook 不进入目标进程

### 用户输入

```text
能看到 module_loaded，但只有主进程日志，目标功能在 com.example.app:remote。Hook 不触发。
```

### 修复前

```java
@Override
public void onPackageReady(XposedModuleInterface.PackageReadyParam param) {
    if (!"com.example.app".equals(param.getPackageName())) {
        return;
    }
    installHook(param);
}
```

### 问题

只判断包名不足以定位多进程场景。目标功能在 `com.example.app:remote`，但日志没有记录进程路由，也没有区分跳过原因。

### 修复后

```java
private static final String TARGET_PACKAGE = "com.example.app";
private static final String TARGET_PROCESS = "com.example.app:remote";

@Override
public void onPackageReady(XposedModuleInterface.PackageReadyParam param) {
    String pkg = param.getPackageName();
    String process = param.getProcessName();

    if (!TARGET_PACKAGE.equals(pkg)) {
        log("LSM-SCOPE-003", "route_skip", pkg, process, "package_mismatch");
        return;
    }
    if (!TARGET_PROCESS.equals(process)) {
        log("LSM-SCOPE-003", "route_skip", pkg, process, "process_mismatch");
        return;
    }

    log("LSM-LOAD-001", "module_loaded", pkg, process, "ok");
    installHook(param);
}
```

### 回答模板

```text
结论：先修进程路由和日志，不扩大 scope。
错误码：LSM-SCOPE-001、LSM-SCOPE-003。
证据：module_loaded 只出现在主进程，目标功能位于 :remote。
修复：增加目标进程判断和 route_skip 日志。
验证：主进程应出现 process_mismatch，remote 进程应出现 module_loaded -> install_hook。
```

## 案例 3：过早使用错误 ClassLoader 导致 ClassNotFound

### 用户输入

```text
module_loaded 有，install_hook 失败，日志是 ClassNotFoundException: com.example.app.FeatureManager。
```

### 修复前

```java
private void installHook(XposedModuleInterface.PackageReadyParam param) throws ClassNotFoundException {
    Class<?> target = Class.forName("com.example.app.FeatureManager");
    hookFeature(target);
}
```

### 问题

`Class.forName()` 默认使用当前模块的类加载上下文，不能保证能找到目标 App 类。Hook 目标类应优先使用 `param.getClassLoader()` 或目标生命周期中确认可用的 ClassLoader。

### 修复后

```java
private void installHook(XposedModuleInterface.PackageReadyParam param) {
    ClassLoader appClassLoader = param.getClassLoader();
    try {
        Class<?> target = Class.forName(
                "com.example.app.FeatureManager",
                false,
                appClassLoader
        );
        hookFeature(target);
        log("LSM-HOOK-001", "install_hook", param.getPackageName(), param.getProcessName(), "ok");
    } catch (ClassNotFoundException e) {
        log("LSM-CL-001", "install_hook", param.getPackageName(), param.getProcessName(), "class_not_found");
    } catch (Throwable t) {
        log("LSM-HOOK-002", "install_hook", param.getPackageName(), param.getProcessName(), "install_failed");
    }
}
```

### 回答模板

```text
结论：这是 ClassLoader 定位问题，先改查找方式和失败日志。
错误码：LSM-CL-001、LSM-HOOK-002。
证据：module_loaded 已出现，但目标类 ClassNotFound。
修复：用目标 App ClassLoader 查找类；找不到时跳过 Hook，不让目标 App 崩溃。
验证：查看 install_hook result=ok；若仍失败，确认目标 App 版本、混淆类名和加载时机。
```

## 案例 4：方法签名不精确导致 Hook 安装成功但不命中

### 用户输入

```text
install_hook 显示成功，但没有 hook_hit。目标方法应该是 setEnabled(boolean)，我只写了方法名。
```

### 修复前

```java
Method method = targetClass.getDeclaredMethod("setEnabled");
method.setAccessible(true);

hook(method)
        .setId("set_enabled_hook")
        .intercept(chain -> {
            log("LSM-HOOK-003", "hook_hit", "setEnabled", "", "ok");
            return chain.proceed();
        });
```

### 问题

目标类可能存在重载方法。只写方法名会让定位不稳定，或者 Hook 到并非实际调用路径的方法。优先确认参数类型和返回值；信息缺失时先列出候选重载和定位步骤。

### 修复后

```java
Method method = targetClass.getDeclaredMethod("setEnabled", boolean.class);
method.setAccessible(true);

hook(method)
        .setId("set_enabled_hook")
        .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
        .intercept(chain -> {
            log("LSM-HOOK-003", "hook_hit", "setEnabled", "boolean", "ok");
            return chain.proceed();
        });
```

### 回答模板

```text
结论：优先确认真实 JVM 签名，不建议扩大 Hook 范围。
错误码：LSM-SIG-001、LSM-HOOK-003。
证据：install_hook 有日志但 hook_hit 没有，且当前只提供方法名。
修复：补充参数类型、返回值和重载判断。
验证：触发目标功能后应出现 hook_hit；若仍无命中，检查调用路径是否走该方法。
```

## 案例 5：Hook 回调直接改返回值导致目标 App 崩溃

### 用户输入

```text
Hook 后目标 App 偶发崩溃。禁用模块后不崩。这里是 before 回调代码。
```

### 修复前

```java
.intercept(chain -> {
    String userId = (String) chain.getArg(0);
    if (userId.startsWith("test")) {
        return true;
    }
    return chain.proceed();
})
```

### 问题

参数可能为 null，也可能不是 String。回调内未做类型校验，异常会影响目标 App。应最小改动：校验参数，失败时 `proceed` 或跳过修改。

### 修复后

```java
.intercept(chain -> {
    Object arg0 = chain.getArg(0);
    if (!(arg0 instanceof String)) {
        log("LSM-HOOK-004", "hook_hit", TARGET_PACKAGE, TARGET_PROCESS, "arg_invalid");
        return chain.proceed();
    }

    String userId = (String) arg0;
    if (userId.startsWith("test")) {
        log("LSM-HOOK-003", "hook_hit", TARGET_PACKAGE, TARGET_PROCESS, "return_skip");
        return true;
    }

    return chain.proceed();
})
```

### 回答模板

```text
结论：这是 Hook 回调缺少参数 Guard，先做防御性修复。
错误码：LSM-HOOK-004、LSM-HOOK-005。
证据：禁用模块后崩溃消失，回调直接强转参数。
修复：增加 null、长度和类型校验；异常或不匹配时 proceed。
验证：空参数、非 String 参数、正常参数和目标功能触发都应不崩溃。
```

## 案例 6：Remote Preferences 类型错误导致配置读取失败

### 用户输入

```text
我用 Remote Preferences 下发 enable_feature，但目标进程读取时偶尔崩溃，日志里有 ClassCastException。
```

### 修复前

```java
boolean enabled = (Boolean) remotePreferences.get("enable_feature");
if (enabled) {
    enableFeature();
}
```

### 问题

远程配置可能缺失、类型错误或读取失败。Hook 进程不能假设配置永远存在且类型正确。默认值必须明确，非法值必须丢弃。

### 修复后

```java
private boolean readEnableFeature() {
    try {
        Object value = remotePreferences.get("enable_feature");
        if (value instanceof Boolean) {
            log("LSM-CFG-001", "read_config", TARGET_PACKAGE, TARGET_PROCESS, "remote_value");
            return (Boolean) value;
        }
        log("LSM-CFG-002", "read_config", TARGET_PACKAGE, TARGET_PROCESS, "invalid_type");
        return false;
    } catch (Throwable t) {
        log("LSM-CFG-001", "read_config", TARGET_PACKAGE, TARGET_PROCESS, "fallback_default");
        return false;
    }
}
```

### 回答模板

```text
结论：配置读取必须有默认值、类型校验和异常回退。
错误码：LSM-CFG-001、LSM-CFG-002。
证据：ClassCastException 来自配置强转。
修复：读取 Object 后判断类型；读取失败或非法类型时使用默认 false。
验证：默认配置、true、false、字符串非法值、远程服务不可用分别测试。
```

## 案例 7：Hot Reload 后旧状态残留导致重复 Hook

### 用户输入

```text
开启 autoHotReload 后，第一次正常，reload 后 hook_hit 变成两次，偶尔状态错乱。
```

### 修复前

```java
private final List<HookHandle> handles = new ArrayList<>();

private void installHook(XposedModuleInterface.PackageReadyParam param) {
    HookHandle handle = hookFeature(param);
    handles.add(handle);
}
```

### 问题

Hot Reload 前没有清理旧状态，也没有确认旧 HookHandle 是否还有效。reload 后重复安装导致多次命中。

### 修复后

```java
private final List<HookHandle> handles = new ArrayList<>();
private final AtomicBoolean installing = new AtomicBoolean(false);

@Override
public void onHotReloading() {
    log("LSM-HR-001", "hot_reload", TARGET_PACKAGE, TARGET_PROCESS, "clean_start");
    for (HookHandle handle : handles) {
        try {
            handle.unhook();
        } catch (Throwable t) {
            log("LSM-HR-002", "hot_reload", TARGET_PACKAGE, TARGET_PROCESS, "unhook_failed");
        }
    }
    handles.clear();
    installing.set(false);
}

private void installHook(XposedModuleInterface.PackageReadyParam param) {
    if (!installing.compareAndSet(false, true)) {
        log("LSM-HOOK-005", "install_hook", param.getPackageName(), param.getProcessName(), "duplicate_skip");
        return;
    }
    HookHandle handle = hookFeature(param);
    handles.add(handle);
}
```

### 回答模板

```text
结论：当前 Hot Reload 不满足状态清理要求，先修清理和防重复安装。
错误码：LSM-HR-001、LSM-HR-002、LSM-HOOK-005。
证据：reload 后 hook_hit 次数翻倍，说明旧 Hook 或状态残留。
修复：onHotReloading 清理 HookHandle 和状态；installHook 增加重复安装保护。
验证：reload 前后 hook_hit 次数保持 1 次；失败时提示重启目标进程。
```

## 案例 8：Native 分支失败时没有降级导致模块整体不可用

### 用户输入

```text
Native Hook 分支在部分设备上加载失败，Java Hook 也不工作了。设备 ABI 是 arm64-v8a。
```

### 修复前

```java
static {
    System.loadLibrary("demo_hook");
}

@Override
public void onPackageReady(XposedModuleInterface.PackageReadyParam param) {
    installNativeHook();
    installJavaHook(param);
}
```

### 问题

Native 分支加载失败不应阻断 Java 分支。Native Hook 应记录 ABI、so 打包、符号和加载时机；信息缺失时先输出检查与观测步骤，失败时禁用 native 分支并保留 Java 分支。

### 修复后

```java
private boolean nativeAvailable;

@Override
public void onPackageReady(XposedModuleInterface.PackageReadyParam param) {
    nativeAvailable = loadNativeBranch();

    if (nativeAvailable) {
        try {
            installNativeHook();
            log("LSM-NATIVE-001", "native_hook", param.getPackageName(), param.getProcessName(), "ok");
        } catch (Throwable t) {
            log("LSM-NATIVE-002", "native_hook", param.getPackageName(), param.getProcessName(), "install_failed");
        }
    }

    installJavaHook(param);
}

private boolean loadNativeBranch() {
    try {
        System.loadLibrary("demo_hook");
        return true;
    } catch (Throwable t) {
        log("LSM-NATIVE-001", "native_load", TARGET_PACKAGE, TARGET_PROCESS, "disabled");
        return false;
    }
}
```

### 回答模板

```text
结论：Native 分支失败应降级，不应阻断 Java 分支。
错误码：LSM-NATIVE-001、LSM-NATIVE-002。
证据：部分 ABI 设备 native 加载失败，Java Hook 也被阻断。
修复：把 native 加载和安装放入可失败分支；失败时记录并继续 Java Hook。
验证：检查 APK 内 lib/<abi>/so、native_init.list、设备 ABI、native 日志和 Java hook_hit。
```

## 输出格式：修复过程回答

当用户给出错误代码、日志或 APK 内容并要求修复时，按下面结构输出：

```text
结论：最可能失败层级和是否可以直接修复。
证据：引用用户给出的日志、文件或代码片段。
错误码：列出本次涉及的 LSM-...。
修复前问题：指出最小问题，不扩大范围。
修复后代码：只给必要片段；保留原业务逻辑。
验证路径：安装、启用 scope、强停目标 App、触发功能、检查日志。
仍需补充：缺少的包名、进程、类名、签名、版本、APK 内容或崩溃堆栈。
```

## 与其他文件配合

- 快速定位失败层级：`guides/troubleshooting-cards.md`。
- 验证和发布前检查：`guides/validation-checklist.md`。
- 错误码和防御性 Guard：`templates/defensive-error-handling.md`。
- Hook API 细节：`knowledge/02-hook-api.md`。
- Remote Preferences / Hot Reload：`knowledge/03-service-remote-hot-reload.md`。
- Native 边界：`guides/special-boundaries.md` + `knowledge/04-native-migration-helper.md`。
