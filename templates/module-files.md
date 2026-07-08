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

如果使用本地 stub：

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

说明：官方 `libxposed/example` 为演示热重载默认使用 `autoHotReload=true`。本模板默认关闭，是为了避免未实现 `onHotReloading()` 清理逻辑时在更新后自动重载；只有完整处理旧 Hook、线程、JNI/native 资源和 `onHotReloaded()` 替换逻辑后，才建议改为 `true`。

只面向 API 102：

```properties
minApiVersion=102
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

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

必须保证：

- 入口类不被删除；
- `java_init.list` 能匹配混淆后的入口类，或入口类不混淆；
- `META-INF/xposed/module.prop` 保留；
- `META-INF/xposed/java_init.list` 保留；
- `META-INF/xposed/scope.list` 保留；
- native 场景下 `native_init.list` 保留。

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