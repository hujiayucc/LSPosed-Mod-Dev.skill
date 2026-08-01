# API 102 完整工程示例

本文件提供一个完整的、可编译的 API 102 模块工程示例，覆盖 Gradle 配置、Java 入口、模块元数据、R8 规则、日志、验证和发布检查清单。与其他案例和模板相比，本文件侧重"开箱即用的完整工程结构"，适合直接复制并调整为真实项目。

## 工程结构

```text
MyLSPosedModule/
├── app/
│   ├── build.gradle
│   ├── proguard-rules.pro
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/example/lspmodule/
│       │   ├── ModuleEntry.java
│       │   └── HookStrategy.java
│       └── resources/META-INF/xposed/
│           ├── module.prop
│           ├── java_init.list
│           └── scope.list
├── build.gradle
├── settings.gradle
└── gradle.properties
```

## 根 build.gradle

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.1.0'
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

## settings.gradle

```groovy
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "MyLSPosedModule"
include ':app'
```

## app/build.gradle

```groovy
plugins {
    id 'com.android.application'
}

android {
    namespace 'com.example.lspmodule'
    compileSdk 34

    defaultConfig {
        applicationId "com.example.lspmodule"
        minSdk 26
        targetSdk 34
        versionCode 1
        versionName "1.0.0"
    }

    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }
}

dependencies {
    compileOnly 'io.github.libxposed:api:102.0.0'
}
```

## app/proguard-rules.pro

```proguard
-dontwarn io.github.libxposed.annotation.**
-adaptresourcefilecontents META-INF/xposed/java_init.list

-keep class * extends io.github.libxposed.XposedModule {
    public <init>(...);
}

-keep class com.example.lspmodule.ModuleEntry {
    public <init>();
}
```

## AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:label="@string/app_name"
        android:description="@string/module_description"
        android:icon="@mipmap/ic_launcher"
        android:allowBackup="false"
        android:extractNativeLibs="false">
    </application>

</manifest>
```

`res/values/strings.xml`：

```xml
<resources>
    <string name="app_name">My LSPosed Module</string>
    <string name="module_description">Example API 102 module</string>
</resources>
```

## META-INF/xposed/module.prop

```properties
id=com.example.lspmodule
name=My LSPosed Module
version=1.0.0
versionCode=1
author=Example
description=API 102 example module
minApiVersion=102
targetApiVersion=102
staticScope=false
exceptionMode=protective
autoHotReload=false
```

## META-INF/xposed/java_init.list

```text
com.example.lspmodule.ModuleEntry
```

## META-INF/xposed/scope.list

```text
com.example.targetapp
```

## ModuleEntry.java

```java
package com.example.lspmodule;

import android.util.Log;
import androidx.annotation.NonNull;
import io.github.libxposed.XposedModule;
import io.github.libxposed.XposedModuleInterface;

public final class ModuleEntry extends XposedModule {
    private static final String TAG = "LSM-Example";
    private static final String TARGET_PACKAGE = "com.example.targetapp";

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

        log(Log.INFO, TAG, "event=route_match result=ok package=" + TARGET_PACKAGE);
        HookStrategy.install(this, param.getClassLoader());
    }
}
```

## HookStrategy.java

```java
package com.example.lspmodule;

import android.util.Log;
import io.github.libxposed.XposedModule;
import io.github.libxposed.XposedInterface;
import java.lang.reflect.Method;
import java.util.concurrent.atomic.AtomicBoolean;

public final class HookStrategy {
    private static final String TAG = "LSM-Example";
    private static final AtomicBoolean installed = new AtomicBoolean(false);

    public static void install(XposedModule module, ClassLoader classLoader) {
        if (!installed.compareAndSet(false, true)) {
            module.log(Log.WARN, TAG, "event=install_hook result=skip reason=already_installed");
            return;
        }

        try {
            Class<?> targetClass = classLoader.loadClass("com.example.targetapp.MainActivity");
            Method targetMethod = targetClass.getDeclaredMethod("exampleMethod", String.class);
            targetMethod.setAccessible(true);

            module.hook(targetMethod)
                    .setId("example_hook_001")
                    .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
                    .intercept(chain -> {
                        Object arg0 = chain.getArg(0);
                        if (!(arg0 instanceof String)) {
                            module.log(Log.WARN, TAG, "event=hook_hit result=skip code=LSM-HOOK-004 reason=bad_arg");
                            return chain.proceed();
                        }

                        String input = (String) arg0;
                        module.log(Log.INFO, TAG, "event=hook_hit result=ok hook_id=example_hook_001 input=" + input);

                        Object original = chain.proceed();
                        module.log(Log.INFO, TAG, "event=hook_return hook_id=example_hook_001 original=" + original);
                        return original;
                    });

            module.log(Log.INFO, TAG, "event=install_hook result=ok hook_id=example_hook_001 target=MainActivity#exampleMethod");

        } catch (ClassNotFoundException e) {
            installed.set(false);
            module.log(Log.WARN, TAG, "event=install_hook result=skip code=LSM-CL-001 reason=class_not_found", e);
        } catch (NoSuchMethodException e) {
            installed.set(false);
            module.log(Log.WARN, TAG, "event=install_hook result=skip code=LSM-SIG-001 reason=method_not_found", e);
        } catch (Throwable t) {
            installed.set(false);
            module.log(Log.ERROR, TAG, "event=install_hook result=fail code=LSM-HOOK-001", t);
        }
    }
}
```

## 构建与验证清单

### 1. 构建 APK

```bash
./gradlew assembleRelease
```

输出：`app/build/outputs/apk/release/app-release.apk`

### 2. 验证 APK 内容

```bash
unzip -l app-release.apk | grep -E 'META-INF/xposed|lib/'
```

预期：
- `META-INF/xposed/module.prop`
- `META-INF/xposed/java_init.list`
- `META-INF/xposed/scope.list`
- 不应包含 `io/github/libxposed/` 类文件（compileOnly 不打包）

### 3. 检查 R8 规则生效

```bash
unzip -l app-release.apk | grep 'com/example/lspmodule/ModuleEntry.class'
```

预期：入口类保留，混淆后仍可被框架找到。

### 4. 安装并启用模块

```bash
adb install -r app-release.apk
# LSPosed Manager 中启用模块并勾选 com.example.targetapp
adb shell am force-stop com.example.targetapp
```

### 5. 检查日志

```bash
adb logcat -s LSM-Example:* | grep -E 'event='
```

预期日志序列：
```text
event=module_loaded result=ok process=com.example.targetapp ...
event=route_match result=ok package=com.example.targetapp
event=install_hook result=ok hook_id=example_hook_001 ...
event=hook_hit result=ok hook_id=example_hook_001 input=...
event=hook_return hook_id=example_hook_001 original=...
```

### 6. 失败场景验证

| 失败点 | 复现方法 | 预期日志 |
|---|---|---|
| 未启用 scope | 不勾选目标 App | `route_skip reason=package_mismatch` |
| 目标类不存在 | 修改类名 | `install_hook result=skip code=LSM-CL-001` |
| 方法签名错误 | 修改方法签名 | `install_hook result=skip code=LSM-SIG-001` |
| Hook 参数类型错误 | 传入非 String | `hook_hit result=skip code=LSM-HOOK-004` |

## 发布前检查表

```text
[ ] module.prop 包含 id、version、minApiVersion=102、targetApiVersion=102、exceptionMode
[ ] java_init.list 指向正确入口类
[ ] scope.list 包含目标包名
[ ] AndroidManifest.xml 包含 android:label 和 android:description
[ ] proguard-rules.pro 包含 libxposed 规则和入口类保留
[ ] dependencies 中 libxposed 为 compileOnly，不是 implementation
[ ] APK 内不包含 io.github.libxposed 类
[ ] APK 内 META-INF/xposed 三个文件齐全
[ ] 日志包含 module_loaded、route_match、install_hook、hook_hit
[ ] 目标类不存在时不崩溃，只记录 skip
[ ] Hook 参数类型不符时回退 chain.proceed()
[ ] 测试覆盖：正常流程、scope 不匹配、类缺失、方法缺失、参数错误
```

## 扩展路径

- 多进程：在 `onPackageReady` 中增加 `processName` 判断和路由表。
- Remote Preferences：添加 `implementation("io.github.libxposed:service:102.0.0")` 并创建配置 App。
- Native Hook：按 [cases/advanced-native-hook.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/cases/advanced-native-hook.md) 补充 CMake、C++ 和 `native_init.list`。
- Hot Reload：只有列清 HookHandle、全局状态和清理策略后才设置 `autoHotReload=true`。

## 与其他文件配合

- API 102 官方参考：[knowledge/07-libxposed-api102-reference.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/knowledge/07-libxposed-api102-reference.md)
- Java 模板：[templates/java-api102.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/templates/java-api102.md)
- Kotlin 模板：[templates/kotlin-api102.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/templates/kotlin-api102.md)
- 模块元数据：[templates/module-files.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/templates/module-files.md)
- 验证清单：[guides/validation-checklist.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/guides/validation-checklist.md)
- 排错卡片：[guides/troubleshooting-cards.md](/data/user/0/com.ai.assistance.operit/files/workspace/LSPosed-Mod-Dev.skill/guides/troubleshooting-cards.md)
