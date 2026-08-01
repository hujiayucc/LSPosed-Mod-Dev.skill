# LSPosed 模块元数据与构建模板

## Gradle 依赖

现代 API 102 默认：

```kotlin
dependencies {
    compileOnly("io.github.libxposed:api:102.0.0")
}
```

如果 Java 模板保留 `androidx.annotation.NonNull`，且项目没有其他 AndroidX 依赖提供该注解，可添加：

```kotlin
dependencies {
    compileOnly("androidx.annotation:annotation:1.9.1")
}
```

如果模块 App 需要与框架通信：

```kotlin
dependencies {
    implementation("io.github.libxposed:service:102.0.0")
}
```
`service` 只在需要 Remote Preferences、Remote Files、scope 查询或 Hot Reload 时加入。

如果使用本地 API stub：

```kotlin
dependencies {
    compileOnly(project(":libxposed-api-stubs"))
}
```

要求：

- `libxposed` API 必须是 `compileOnly`；
- `androidx.annotation` 只用于编译期注解时也建议使用 `compileOnly`；
- 不得用 `implementation` 把 API 打包进 APK；
- release 混淆必须保留入口类和 `META-INF/xposed` 元数据；
- 需要 service 时再加入 `libxposed/service`；
- 需要 helper 时再加入 `libxposed/helper`，并先确认依赖版本可解析。

## module.prop

默认模板：

```properties
minApiVersion=101
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

说明：

- `minApiVersion` 和 `targetApiVersion` 是必填字段；只使用 API 102 且不兼容 101 时可将两者都设为 `102`；
- `staticScope` 表示 scope 是否固定；
- `exceptionMode` 只接受 `protective` 或 `passthrough`，Java Hook 侧使用 `ExceptionMode.DEFAULT` 跟随该配置；
- `autoHotReload` 是 API 102+ 选项；只有实现 `onHotReloading()` / `onHotReloaded()`、旧 Hook 清理和替换后再打开。

只面向 API 102：

```properties
minApiVersion=102
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

模块名称与描述来自 AndroidManifest 的标准资源：

```xml
<application
    android:label="@string/app_name"
    android:description="@string/app_description">
</application>
```

旧 `xposedmodule`、`xposeddescription` 和 `xposedminversion` 不作为新模块默认配置。

## java_init.list

```text
com.example.module.ModuleEntry
```

要求：

- 每行一个入口类；
- 类名必须完整；
- 不要多余空格；
- Hot Reload 场景最好只有一个 Java 入口；
- 混淆时必须保证入口类可被 LSPosed 定位。

## scope.list

普通 App：

```text
com.example.target
```

SystemUI 场景：

```text
com.example.target
com.android.systemui
```

system_server 场景：

```text
system
com.example.target
```

要求：

- scope 必须最小；
- `system` 和 `com.android.systemui` 必须有明确理由；
- 不要为了保险扩大 scope；
- 修改 scope 后通常需要重启或强停目标应用。

## ProGuard / R8

官方 API README 的基础规则：

```proguard
-dontwarn io.github.libxposed.annotation.**
-adaptresourcefilecontents META-INF/xposed/java_init.list
-keep,allowoptimization,allowobfuscation public class * extends io.github.libxposed.api.XposedModule {
    public <init>();
}
```

这些规则用于保留入口类和无参构造，并在入口类混淆时同步改写 `java_init.list`。同时确认 `module.prop`、`scope.list` 和 Native 场景下的 `native_init.list` 被打包。

## 验证 APK

构建后应检查：

```text
META-INF/xposed/module.prop
META-INF/xposed/java_init.list
META-INF/xposed/scope.list
```

Native Hook 场景还应检查：

```text
META-INF/xposed/native_init.list
lib/<abi>/libexample.so
```

如果缺失，模块不会正常加载、scope 不生效，或 native 入口不会被框架识别。