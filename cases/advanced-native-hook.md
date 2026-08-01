# 高级 Native Hook 示例

本文件补充 `knowledge/04-native-migration-helper.md` 中的 native_init 基础说明，提供一个可审查的高级示例骨架：Java 入口先做包名、进程和风险判断，再按需加载 native so；C++ 层实现 `native_init`、延迟 so 回调、符号查找、Hook 安装、backup 调用和失败降级。

本示例面向 APP 静态分析、动态调试、兼容性定位和 LSPosed Native Hook 工程。涉及系统稳定性、数据处理或高影响路径时，必须保留版本记录、最小 scope、崩溃保护、日志脱敏和回滚方案。

## 前置条件

生成真实 Native Hook 前必须确认：

```text
analysis_scope=<分析范围或测试环境>
target_package=<目标包名>
target_process=<目标进程>
target_so=<目标 so 名称>
target_symbol=<目标符号或可稳定定位方式>
abi=<arm64-v8a|armeabi-v7a|x86_64|x86>
function_signature=<返回值和参数>
fallback=<native 失败时 Java 分支如何继续>
crash_plan=<native crash 日志或 tombstone 获取方式>
```

缺任一项时，不生成最终 Hook，只给定位步骤和验证清单。

## 文件结构

```text
app/src/main/java/com/example/module/ModuleEntry.java
app/src/main/cpp/native_hook.cpp
app/src/main/cpp/CMakeLists.txt
app/src/main/resources/META-INF/xposed/module.prop
app/src/main/resources/META-INF/xposed/java_init.list
app/src/main/resources/META-INF/xposed/native_init.list
app/build.gradle
```

## Gradle / CMake 配置片段

`app/build.gradle` 示例：

```groovy
android {
    namespace "com.example.module"

    defaultConfig {
        minSdk 26
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++17 -fvisibility=hidden"
                arguments "-DANDROID_STL=c++_static"
            }
        }
        ndk {
            abiFilters "arm64-v8a"
        }
    }

    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
        }
    }
}

dependencies {
    compileOnly "io.github.libxposed:api:102.0.0"
}
```

`CMakeLists.txt` 示例：

```cmake
cmake_minimum_required(VERSION 3.22.1)
project(lsp_native_demo)

add_library(lsp_native_demo SHARED native_hook.cpp)

target_compile_features(lsp_native_demo PRIVATE cxx_std_17)
target_link_libraries(lsp_native_demo PRIVATE log dl)
```

`native_init.list`：

```text
liblsp_native_demo.so
```

注意：`native_init.list` 中写 APK 内 so 文件名；Java 层 `System.loadLibrary("lsp_native_demo")` 使用不带 `lib` 前缀和 `.so` 后缀的库名。

## Java 入口加载策略

Java 层负责先收窄目标，再加载 native 分支。Native 加载失败时必须保留 Java 分支和目标 App 原流程。

```java
public final class ModuleEntry extends XposedModule {
    private static final String TAG = "LSM-NativeDemo";
    private static final String TARGET_PACKAGE = "com.example.app";
    private static volatile boolean nativeAvailable;

    @Override
    public void onModuleLoaded(@NonNull XposedModuleInterface.ModuleLoadedParam param) {
        log(Log.INFO, TAG, "event=module_loaded result=ok process=" + param.getProcessName()
                + " api=" + getApiVersion()
                + " framework=" + getFrameworkName()
                + " version=" + getFrameworkVersion());
    }

    @Override
    public void onPackageReady(@NonNull XposedModuleInterface.PackageReadyParam param) {
        if (!TARGET_PACKAGE.equals(param.getPackageName())) {
            log(Log.INFO, TAG, "event=route_skip result=skip reason=package_mismatch package=" + param.getPackageName());
            return;
        }
        if (!param.isFirstPackage()) {
            log(Log.INFO, TAG, "event=route_skip result=skip reason=not_first_package package=" + param.getPackageName());
            return;
        }

        installJavaFallbackHook(param.getClassLoader());
        loadNativeBranch();
    }

    private void loadNativeBranch() {
        try {
            System.loadLibrary("lsp_native_demo");
            nativeAvailable = true;
            log(Log.INFO, TAG, "event=native_load result=ok lib=liblsp_native_demo.so");
        } catch (UnsatisfiedLinkError e) {
            nativeAvailable = false;
            log(Log.ERROR, TAG, "event=native_load result=fail code=LSM-NATIVE-001 recover=java_branch", e);
        } catch (Throwable t) {
            nativeAvailable = false;
            log(Log.ERROR, TAG, "event=native_load result=fail code=LSM-NATIVE-002 recover=java_branch", t);
        }
    }

    private void installJavaFallbackHook(ClassLoader classLoader) {
        try {
            Class<?> bridge = classLoader.loadClass("com.example.app.NativeBridge");
            Method method = bridge.getDeclaredMethod("check", String.class);
            hook(method).intercept(chain -> {
                Object arg0 = chain.getArg(0);
                if (!(arg0 instanceof String)) {
                    log(Log.WARN, TAG, "event=hook_hit result=skip code=LSM-HOOK-004 reason=bad_arg");
                    return chain.proceed();
                }
                Object original = chain.proceed();
                log(Log.INFO, TAG, "event=hook_hit result=ok branch=java_fallback native=" + nativeAvailable);
                return original;
            });
            log(Log.INFO, TAG, "event=install_hook result=ok branch=java_fallback");
        } catch (ClassNotFoundException e) {
            log(Log.WARN, TAG, "event=install_hook result=skip code=LSM-CL-001 branch=java_fallback", e);
        } catch (NoSuchMethodException e) {
            log(Log.WARN, TAG, "event=install_hook result=skip code=LSM-SIG-001 branch=java_fallback", e);
        } catch (Throwable t) {
            log(Log.ERROR, TAG, "event=install_hook result=fail code=LSM-HOOK-001 branch=java_fallback", t);
        }
    }
}
```

要点：

- 不在无关包或无关进程加载 native 库；
- Java Hook 先作为最小可观测路径；
- `System.loadLibrary()` 失败只禁用 native 分支，不让目标 App 崩溃；
- `nativeAvailable` 只作为状态标记，不作为安全判断依据。

## C++ native_init 示例

示例目标：目标 so `libdemo.so` 加载后，查找符号 `demo_check`，安装 replacement。真实项目必须替换函数签名和符号名。

```cpp
#include <android/log.h>
#include <dlfcn.h>
#include <stdint.h>
#include <string.h>
#include <atomic>

#define LOG_TAG "LSM-NativeDemo"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN, LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

using HookFunType = int (*)(void *func, void *replace, void **backup);
using UnhookFunType = int (*)(void *func);
using NativeOnModuleLoaded = void (*)(const char *name, void *handle);

struct NativeAPIEntries {
    uint32_t version;
    HookFunType hook_func;
    UnhookFunType unhook_func;
};

using DemoCheck = int (*)(const char *value);

static HookFunType g_hook_func = nullptr;
static UnhookFunType g_unhook_func = nullptr;
static DemoCheck g_backup_demo_check = nullptr;
static std::atomic_bool g_hooked{false};

static int replacement_demo_check(const char *value) {
    if (value == nullptr) {
        LOGW("event=native_hook_hit result=skip reason=null_arg recover=backup");
        return g_backup_demo_check != nullptr ? g_backup_demo_check(value) : 0;
    }

    if (g_backup_demo_check == nullptr) {
        LOGE("event=native_hook_hit result=fail reason=missing_backup recover=default");
        return 0;
    }

    int original = g_backup_demo_check(value);
    LOGI("event=native_hook_hit result=ok original=%d", original);
    return original;
}

static void on_library_loaded(const char *name, void *handle) {
    if (name == nullptr || handle == nullptr) {
        LOGW("event=native_library_loaded result=skip reason=null_input");
        return;
    }
    if (strstr(name, "libdemo.so") == nullptr) {
        return;
    }
    if (g_hook_func == nullptr) {
        LOGE("event=native_install result=fail code=LSM-NATIVE-003 reason=missing_hook_func");
        return;
    }

    bool expected = false;
    if (!g_hooked.compare_exchange_strong(expected, true)) {
        LOGI("event=native_install result=skip reason=already_hooked target=demo_check");
        return;
    }

    void *symbol = dlsym(handle, "demo_check");
    if (symbol == nullptr) {
        g_hooked.store(false);
        LOGW("event=native_install result=skip code=LSM-NATIVE-004 reason=symbol_missing target=demo_check");
        return;
    }

    void *backup = nullptr;
    int rc = g_hook_func(symbol, reinterpret_cast<void *>(replacement_demo_check), &backup);
    if (rc != 0 || backup == nullptr) {
        g_hooked.store(false);
        LOGE("event=native_install result=fail code=LSM-NATIVE-005 rc=%d backup=%p", rc, backup);
        return;
    }

    g_backup_demo_check = reinterpret_cast<DemoCheck>(backup);
    LOGI("event=native_install result=ok target=demo_check backup=%p", backup);
}

extern "C" [[gnu::visibility("default")]] [[gnu::used]]
NativeOnModuleLoaded native_init(const NativeAPIEntries *entries) {
    if (entries == nullptr) {
        LOGE("event=native_init result=fail code=LSM-NATIVE-006 reason=null_entries");
        return nullptr;
    }
    if (entries->hook_func == nullptr || entries->unhook_func == nullptr) {
        LOGE("event=native_init result=fail code=LSM-NATIVE-007 reason=missing_entries version=%u", entries->version);
        return nullptr;
    }

    g_hook_func = entries->hook_func;
    g_unhook_func = entries->unhook_func;
    LOGI("event=native_init result=ok version=%u", entries->version);
    return on_library_loaded;
}
```

## Native 错误码

| 错误码 | 含义 | 回退 |
|---|---|---|
| LSM-NATIVE-001 | `System.loadLibrary()` 找不到 so 或 ABI 不匹配 | 禁用 native 分支 |
| LSM-NATIVE-002 | Java 层 native 加载出现其他异常 | 禁用 native 分支 |
| LSM-NATIVE-003 | `entries->hook_func` 不可用 | 跳过 native Hook |
| LSM-NATIVE-004 | `dlsym()` 找不到目标符号 | 跳过 native Hook |
| LSM-NATIVE-005 | `hook_func()` 失败或 backup 为空 | 撤销安装状态 |
| LSM-NATIVE-006 | `native_init` 收到空 entries | 不返回 callback |
| LSM-NATIVE-007 | `NativeAPIEntries` 缺必要函数 | 不返回 callback |

## 验证路径

1. APK 内存在 `META-INF/xposed/native_init.list`；
2. APK 内存在 `lib/arm64-v8a/liblsp_native_demo.so` 或目标 ABI；
3. `native_init` 导出且未被裁剪；
4. Java 日志出现 `event=native_load result=ok`；
5. Native 日志出现 `event=native_init result=ok`；
6. 目标 so 加载后出现 `event=native_install result=ok`；
7. 触发目标函数后出现 `event=native_hook_hit result=ok`；
8. 失败时 Java 分支仍有 `branch=java_fallback` 日志，目标 App 不崩溃。

## 失败降级矩阵

| 失败点 | 处理 |
|---|---|
| ABI 不匹配 | 禁用 native 分支，提示检查 `abiFilters` 和 APK lib 目录 |
| so 未打包 | 修复 CMake/Gradle 和 `native_init.list` |
| `native_init` 未导出 | 检查 `extern "C"`、visibility、strip 配置 |
| 目标 so 未加载 | 等待 callback，不主动轮询 |
| 符号缺失 | 跳过该符号，记录目标版本 |
| backup 为空 | 撤销安装状态，不调用 replacement 逻辑 |
| native crash | 回退 Java 分支，要求 tombstone 和 ABI 信息 |

## 与其他文件配合

- Native 基础：`knowledge/04-native-migration-helper.md`。
- 高风险边界：`guides/special-boundaries.md`。
- 混合模块拆解：`guides/advanced-combinations.md`。
- 验证清单：`guides/validation-checklist.md`。
- 真实 API 102 场景：`cases/api102-real-cases.md`。