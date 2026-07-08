# 01 - 工程基础与模块元数据

覆盖原知识库第 1-13 章：角色定义、安全边界、默认技术路线、依赖配置、ProGuard/R8、工程结构、Gradle、Manifest、`java_init.list`、`module.prop`、`scope.list`、入口类和生命周期。

## 默认角色与边界

本 Skill 只面向合法、授权、学习型、调试型和兼容适配型 LSPosed 模块开发。默认技术路线是 Modern Xposed API，也就是 `libxposed/api`，默认目标 API 为 102。

可以协助：

- 创建现代 LSPosed 模块工程；
- 编写 `XposedModule` 入口类；
- 配置 `META-INF/xposed/java_init.list`、`module.prop`、`scope.list`；
- 排查模块不加载、Hook 不触发、ClassLoader、scope、签名和生命周期问题；
- 使用 `libxposed/service`、Remote Preferences、Remote Files；
- 迁移旧 Xposed 模块到现代 API；
- 审查 LSPosed 模块质量。

必须拒绝或改写：绕过检测、反作弊对抗、隐藏模块、隐蔽注入、未授权修改第三方 App、窃取隐私或凭据、破坏系统稳定、持久化后门。遇到危险请求时只可转为合法学习、稳定性排错、兼容性修复或防护分析。

## 默认技术路线

优先使用：

- `io.github.libxposed.api.XposedModule`；
- `META-INF/xposed/java_init.list`；
- `META-INF/xposed/module.prop`；
- `META-INF/xposed/scope.list`；
- `hook(Executable)`；
- `XposedInterface.Chain`；
- `XposedInterface.Hooker`；
- `XposedInterface.Invoker`；
- `libxposed/service`。

除非用户明确维护旧模块，否则不要默认使用 `IXposedHookLoadPackage`、`XposedHelpers`、`XC_MethodHook`、`assets/xposed_init` 或资源 Hook。

## 依赖配置

Hook 入口依赖使用 `compileOnly`：

```kotlin
dependencies {
    compileOnly("io.github.libxposed:api:102.0.0")
}
```

如果模块 App 需要与框架通信，添加 service 依赖：

```kotlin
dependencies {
    implementation("io.github.libxposed:service:102.0.0")
}
```

Java 模板如保留 `androidx.annotation.NonNull`，且项目没有其他 AndroidX 注解依赖，可添加：

```kotlin
dependencies {
    compileOnly("androidx.annotation:annotation:1.9.1")
}
```

`libxposed/helper` 是可选辅助库，只在复杂查找、混淆定位、Dex 特征匹配时建议使用。生成真实工程前必须先确认 Maven/Gradle 可解析版本，不要把 helper 当成 `api:102.0.0` 的同版本组件。

## ProGuard / R8

推荐规则：

```proguard
-dontwarn io.github.libxposed.annotation.**
-adaptresourcefilecontents META-INF/xposed/java_init.list
-keep,allowoptimization,allowobfuscation public class * extends io.github.libxposed.api.XposedModule {
    public <init>();
}
```

关键点：入口类必须保留无参构造；如果入口类被混淆，必须同步重写 `java_init.list`；错误混淆规则会导致模块无法加载。

## 工程结构

现代模块推荐结构：

```text
app/src/main/java/<your/package>/ModuleMain.java
app/src/main/resources/META-INF/xposed/java_init.list
app/src/main/resources/META-INF/xposed/module.prop
app/src/main/resources/META-INF/xposed/scope.list
app/src/main/AndroidManifest.xml
app/build.gradle.kts
app/proguard-rules.pro
```

Native Hook 额外需要：

```text
app/src/main/resources/META-INF/xposed/native_init.list
```

Gradle 重点：`META-INF/xposed/*` 必须打包进 APK；官方 example 常用 `packaging.resources.merges += "META-INF/xposed/*"`。如果真实模块有 UI、图片、raw 或其他运行时资源，不要盲目使用 `packaging.resources.excludes += "**"`。

## AndroidManifest

现代 API 不依赖旧 Xposed metadata。模块名称和描述来自 Android 标准资源：

```xml
<application
    android:label="@string/app_name"
    android:description="@string/app_description">
</application>
```

新模块不要默认使用旧的 `xposedmodule`、`xposeddescription`、`xposedminversion`。

## java_init.list

路径：

```text
app/src/main/resources/META-INF/xposed/java_init.list
```

内容示例：

```text
your.package.name.ModuleMain
```

规则：每行一个入口类完整类名；入口类必须继承 `XposedModule`；Hot Reload 要求模块声明恰好一个 Java 入口类；现代模块不再使用 `assets/xposed_init`。

## module.prop

路径：

```text
app/src/main/resources/META-INF/xposed/module.prop
```

推荐配置：

```properties
minApiVersion=101
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

说明：只使用 API 102 且不兼容 101 时可设 `minApiVersion=102`；`targetApiVersion >= 102` 时不能调用 legacy `de.robv.android.xposed` API；只有明确支持热重载时才设置 `autoHotReload=true`。

## scope.list

路径：

```text
app/src/main/resources/META-INF/xposed/scope.list
```

普通 App：

```text
com.example.target
```

system server：

```text
system
```

规则：scope 越小越好；不要为了保险扩大 scope；涉及 `system`、`com.android.systemui` 或 framework 路径时必须说明理由和风险。

## 入口类职责

入口类应负责：记录模块加载、判断 API/框架/包名/进程、选择生命周期、路由到 Hook Installer、处理 Hot Reload 生命周期和必要的 service 监听。

入口类不应负责：堆积大量反射逻辑、直接写复杂业务、在所有进程盲目安装 Hook、捕获异常后静默吞掉、未说明原因地混用旧 API。

## 生命周期选择

常用生命周期：

- `onModuleLoaded()`：模块加载后调用，适合记录框架信息、初始化模块状态；
- `onPackageLoaded()`：目标包加载时调用，适合早期 Hook；
- `onPackageReady()`：API 101+，目标包更完整可用时调用，普通 App Hook 优先考虑；
- `onSystemServerStarting()`：system_server 相关路径，高风险，必须谨慎；
- `onHotReloading()` / `onHotReloaded()`：API 102 Hot Reload 生命周期。

普通 App Hook 默认优先在 `onPackageReady()` 中使用 `param.getClassLoader()` 定位目标类。不要在构造函数或 `onModuleLoaded()` 中过早查找目标 App 类。