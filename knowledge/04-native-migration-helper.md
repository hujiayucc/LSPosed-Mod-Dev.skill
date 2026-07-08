# 04 - Native Hook、旧 API 迁移与 helper

覆盖原知识库第 28-31 章：Native Hook、旧 API 兼容与迁移、`libxposed/helper`、日志规范。

## Native Hook 边界

只有用户明确需要 Native Hook 时才展开。Native Hook 风险明显高于普通 Java Hook，默认不要为普通需求生成 Native Hook。

要求用户具备：NDK、C/C++、so 加载、JNI、native crash 排查、ABI 与符号验证能力。

如果需求可以通过 Java Hook、公开 framework API 或模块 App 配置完成，优先不用 Native Hook。

## native_init

入口函数：

```cpp
extern "C" [[gnu::visibility("default")]] [[gnu::used]]
NativeOnModuleLoaded native_init(const NativeAPIEntries *entries);
```

规则：函数名必须是 `native_init`；必须导出；使用 `extern "C"` 避免 C++ name mangling；使用 `[[gnu::visibility("default")]]` 和 `[[gnu::used]]`；不要修改 `NativeAPIEntries`；已加载系统库函数可在 `native_init` 中 Hook；后续 `dlopen()` 的 so 应在返回 callback 中按库名判断后再 Hook。

## NativeAPIEntries

```cpp
typedef int (*HookFunType)(void *func, void *replace, void **backup);
typedef int (*UnhookFunType)(void *func);
typedef void (*NativeOnModuleLoaded)(const char *name, void *handle);

typedef struct {
    uint32_t version;
    HookFunType hook_func;
    UnhookFunType unhook_func;
} NativeAPIEntries;
```

使用要求：

- 检查 `entries`、`entries->hook_func`、`entries->unhook_func` 是否可用；
- 检查 `dlsym()` 返回值，找不到符号时记录并跳过；
- 检查 `hook_func(...)` 返回值；
- 保存 backup 指针，替换函数中需要调用原函数时使用 backup；
- 防止重复 Hook 同一函数；
- 认识到目标符号可能被 strip、隐藏、inline 或版本变动。

## Native 最小示例要点

最小 C++ 示例应包含：

- `#include <dlfcn.h>`、`stdint.h`、必要 C/C++ 头文件；
- `HookFunType`、`UnhookFunType`、`NativeOnModuleLoaded`、`NativeAPIEntries` 定义；
- 全局保存 `hook_func` 与 backup 指针；
- replacement 函数签名必须与目标函数一致；
- `on_library_loaded()` 中检查 `name`、`handle`、`hook_func` 和重复 Hook 标记；
- 用 `dlsym(handle, "target_func")` 查找符号；
- `hook_func(symbol, replacement, &backup)` 成功后标记已 Hook；
- `native_init()` 检查 entries 后返回 callback。

生成真实代码时必须替换目标 so 名称、目标函数名、目标函数签名、replacement 返回值和参数、重复 Hook 保护策略、日志和错误处理方式。

## native_init.list

现代 API 应使用：

```text
app/src/main/resources/META-INF/xposed/native_init.list
```

内容示例：

```text
libexample.so
```

规则：每行写模块 APK 内包含 `native_init` 的 so 名称；Wiki 中旧示例可能写 `assets/native_init`，现代 API 应优先使用 `META-INF/xposed/native_init.list`；加载 native 库仍需在合适时机 `System.loadLibrary()`；通常应在 Java/Kotlin 入口完成包名、进程和风险判断后再加载 native 库；不要在无关进程加载 native 库。

## ABI 与 JNI 检查

Native Hook 发布前检查：

1. APK 内存在 `META-INF/xposed/native_init.list`；
2. APK 内存在 `lib/<abi>/libexample.so`；
3. 目标进程是 32 位还是 64 位；
4. APK 是否提供目标进程需要的 ABI；
5. Gradle `abiFilters` 是否误删目标 ABI；
6. `native_init.list` 中的 so 名称是否与 APK 内文件名一致；
7. release 构建是否 strip 了模块自身必须导出的 `native_init`；
8. 目标符号在目标版本中是否存在。

JNI 规则：`JNIEnv *` 是线程相关对象，不应跨线程缓存；跨线程使用 JNI 应缓存 `JavaVM *`，并在线程内重新获取 `JNIEnv *`；native 热重载不会自动调用 `JNI_OnUnload`，不保证 `dlclose`；旧 native 线程、回调、JNI 全局引用未清理会造成崩溃。

## Native Hook 排错顺序

1. 是否真的需要 Native Hook，能否改用 Java Hook；
2. `native_init.list` 是否打进 APK；
3. so 是否存在于目标 ABI 路径；
4. `System.loadLibrary()` 是否在目标包和目标进程中执行；
5. `native_init` 是否导出且未被裁剪；
6. `entries->hook_func` 是否为空；
7. callback 是否被目标 so 加载触发；
8. `name` 是否匹配目标 so；
9. `dlsym()` 是否找到目标符号；
10. `hook_func(...)` 返回值是否成功；
11. replacement 签名是否与目标函数完全一致；
12. backup 是否正确保存并调用；
13. 是否重复 Hook；
14. 是否有 native crash 日志和 tombstone；
15. 是否涉及 Hot Reload 未清理 native 状态。

## 旧 API 兼容与迁移

旧模块常用 `IXposedHookLoadPackage` 和 `assets/xposed_init`。现代模块应使用 `XposedModule` 和 `META-INF/xposed/java_init.list`。

旧 API 常见工具：

- `XposedHelpers.findClass`；
- `XposedHelpers.findAndHookMethod`；
- `XposedHelpers.findAndHookConstructor`；
- `XposedHelpers.getObjectField`；
- `XposedHelpers.setObjectField`；
- `XposedHelpers.callMethod`；
- `XposedHelpers.callStaticMethod`。

现代替代方式：直接用 Java 反射、`hook(Executable)`、`getInvoker(Method)`、`getInvoker(Constructor)`，必要时可选 `libxposed/helper`。

迁移规则：旧模块迁移时可以先保留旧 Hook 逻辑，只替换入口和元数据；兼容层必须标注为临时迁移方案；多入口兼容会增加复杂度，只有用户明确需要兼容旧 LSPosed 时才生成。

## 资源 Hook 与 XSharedPreferences

旧 API 支持资源 Hook，现代 API 不支持。不要建议用户用资源 Hook；如果需求是 UI 修改，应考虑普通 Android 资源、运行时 View Hook 或模块 App 配置。

旧 `XSharedPreferences` 只读、不支持 edit、原版不支持 listener、需要 `reload()`，且可能受 SELinux、权限和存储路径影响。现代模块优先使用 Remote Preferences 和 Remote Files。

## libxposed/helper

`libxposed/helper` 是现代辅助库，用于更方便地查找类、方法、字段、构造器、参数、字符串、返回类型、修饰符和调用关系。

适合场景：目标类名稳定但方法签名复杂；方法被混淆但包含稳定字符串；需要按字段类型、调用方法或 Dex 特征定位目标。

Java 入口示意：

```java
HookBuilder.buildHooks(ctx, classLoader, sourcePath, builder -> {
    builder.firstMethod(matcher -> {
        // matcher rules
    }).onMatch(method -> {
        ctx.hook(method).intercept(chain -> chain.proceed());
    });
});
```

注意：`buildHooks` 返回 `Future<?>`；构建可能异步执行；Dex 分析耗时，复杂匹配应使用缓存；不要在主线程做重度分析。

helper 可匹配类、方法、字段、构造器和 Dex 特征。`onMatch` 用于找到目标后安装 Hook，`onMiss` 用于记录找不到目标或切换备用匹配。Reflector 支持 Java 风格和 Dex 风格签名，并会缓存反射结果。

## 日志规范

应使用 Xposed 日志：

```java
log(Log.INFO, TAG, "message");
log(Log.ERROR, TAG, "failed", throwable);
```

不要只用 Android `Log.d(...)`。

建议记录：模块入口加载、每个生命周期、每个 Hook 成功、每个异常堆栈、`packageName`、`processName`、`classLoader`、framework name/version/api、capability。

推荐结构化日志格式：

```text
<Module>/<Component>: event=<事件> package=<包名> process=<进程> reason=<原因>
```

常用事件：`module_loaded`、`install_started`、`install_skipped`、`hook_registered`、`hook_hit`、`condition_skipped`、`class_not_found`、`method_not_found`、`config_loaded`、`fallback_to_original`、`exception_caught`。