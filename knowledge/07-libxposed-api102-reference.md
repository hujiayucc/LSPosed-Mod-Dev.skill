# 07 - libxposed API 102 官方参考

本分片以 `libxposed/api` 的 README、Javadoc 和官方 example 为准，用来校正模板、生命周期、Hook 链、模块元数据和逆向分析路由。它是 API 事实参考，不替代按任务读取的实现模板。

## 官方来源

- API README：<https://github.com/libxposed/api>
- API 包说明：<https://github.com/libxposed/api/blob/master/api/src/main/java/io/github/libxposed/api/package-info.java>
- `XposedInterface`：<https://github.com/libxposed/api/blob/master/api/src/main/java/io/github/libxposed/api/XposedInterface.java>
- `XposedModuleInterface`：<https://github.com/libxposed/api/blob/master/api/src/main/java/io/github/libxposed/api/XposedModuleInterface.java>
- 官方示例：<https://github.com/libxposed/example>
- Service README：<https://github.com/libxposed/service>

生成代码前优先核对这些来源；本地其他分片与上游 API 名称冲突时，以源码和 Javadoc 为准。

## 集成基线

模块 Hook 侧使用编译期依赖：

```kotlin
dependencies {
    compileOnly("io.github.libxposed:api:102.0.0")
}
```

需要模块 App 与框架通信时，在模块 App 侧使用：

```kotlin
dependencies {
    implementation("io.github.libxposed:service:102.0.0")
}
```

`api` 由框架在运行时提供，不应作为 `implementation` 打进模块 APK。`service` 只在需要模块 App 通信、Remote Preferences、Remote Files、scope 查询或 Hot Reload 触发时加入。

## 入口与元数据

Java 入口类继承 `io.github.libxposed.api.XposedModule`。框架会自动连接内部 framework bridge；模块不调用 `attachFramework()`，也不要在 `onModuleLoaded()` 之前执行依赖目标 App ClassLoader 的初始化。

资源目录：

```text
app/src/main/resources/META-INF/xposed/java_init.list
app/src/main/resources/META-INF/xposed/native_init.list
app/src/main/resources/META-INF/xposed/module.prop
app/src/main/resources/META-INF/xposed/scope.list
```

- `java_init.list`：每行一个完整 Java 入口类名；现代模块至少提供一个 Java 入口。
- `native_init.list`：Native 入口类或函数使用的 Native 注册清单，只有 Native 模块需要。
- `scope.list`：每行一个包名；system_server 使用特殊 scope `system`。
- 模块名称和描述使用 Android 标准资源 `android:label` 与 `android:description`，不再依赖旧 Xposed metadata。

`module.prop` 的字段：

```properties
minApiVersion=101
targetApiVersion=102
staticScope=true
exceptionMode=protective
autoHotReload=false
```

- `minApiVersion`、`targetApiVersion` 必填。
- `staticScope` 可选，表示 scope 是否固定。
- `exceptionMode` 可选值为 `protective` 或 `passthrough`；它是模块级默认异常处理配置。
- `autoHotReload` 是 API 102+ 可选项，表示应用更新时是否尝试自动热重载；旧代码的 `onHotReloading()` 仍决定是否继续。

启用 R8/ProGuard 时，官方 README 的核心规则是：

```proguard
-dontwarn io.github.libxposed.annotation.**
-adaptresourcefilecontents META-INF/xposed/java_init.list
-keep,allowoptimization,allowobfuscation public class * extends io.github.libxposed.api.XposedModule {
    public <init>();
}
```

## 生命周期与 ClassLoader

| 回调 | 适用时机 | 关键事实 |
|---|---|---|
| `onModuleLoaded()` | 模块 generation 加载到进程后 | 记录 API、框架、进程和模块状态；不要在此阶段查找目标 App 类 |
| `onPackageLoaded()` | 默认 ClassLoader 已就绪、`AppComponentFactory` 实例化前 | API 29+ 可用于早期定位和早期 Hook |
| `onPackageReady()` | AppComponentFactory 已创建目标 ClassLoader 后 | 普通 App Hook 通常从这里开始，使用 `param.getClassLoader()` |
| `onSystemServerStarting()` | system_server 启动阶段 | 使用 `param.getClassLoader()`；单独处理 system_server 路径 |
| `onHotReloading()` | 旧 generation 即将冻结 | 返回 `true` 才继续；只传递 classloader-neutral 状态 |
| `onHotReloaded()` | 新 generation 加载后 | 不会自动重放包生命周期回调；使用旧 handle 替换或取消 Hook |

`PackageLoadedParam` 与 `PackageReadyParam` 的回调可能针对进程中加载的多个包触发，而不仅是原始 scope 包。因此入口应同时过滤 `packageName` 和 `processName`；不需要继续接收回调时可以调用 `detach()`。普通 App 的目标类查找优先使用 `PackageReadyParam.getClassLoader()`，不要使用模块自身的 ClassLoader。

## Hook 链模型

现代 API 通过 `XposedInterface.hook(Executable)` 返回 `HookBuilder`，再由 `intercept(Hooker)` 注册拦截器：

```java
HookHandle handle = hook(method)
        .setId("target_method")
        .setPriority(XposedInterface.PRIORITY_DEFAULT)
        .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
        .intercept(chain -> {
            Object value = chain.getArg(0);
            Object result = chain.proceed();
            return result;
        });
```

`Chain` 的实际方法包括：

```java
chain.getExecutable();
chain.getThisObject();
chain.getArgs();
chain.getArg(index);
chain.proceed();
chain.proceed(newArgs);
chain.proceedWith(newThis);
chain.proceedWith(newThis, newArgs);
```

- `getArgs()` 返回不可变列表；修改参数时复制为数组后调用 `proceed(Object[])`。
- 静态方法的 `getThisObject()` 为 `null`。
- `Chain` 不能缓存、跨线程传递或在 `intercept()` 返回后继续使用。
- 不调用 `proceed()` 会直接返回当前拦截器给出的结果；观测型 Hook 应调用一次并返回原结果。
- `ExceptionMode.DEFAULT` 跟随 `module.prop` 的全局配置；`PASSTHROUGH` 用于需要让 Hooker 异常继续传播的调试场景。Java 枚举没有 `PROTECTIVE` 常量。

`HookHandle` 提供 `unhook()`、`getId()`、`getExecutable()` 和 API 102 的 `replaceHook(Hooker)`。带 id 的 Hook 可以按同一 executable 和 id 原子替换；替换不影响已经进入的调用，旧 handle 在成功替换后失效。

## Invoker、deoptimize 与资源 Hook

- `getInvoker(Method)` 和 `getInvoker(Constructor)` 返回绕过访问检查的 Invoker。
- `Invoker.Type.ORIGIN` 跳过所有 Hook，`Invoker.Type.Chain.FULL` 走完整链，`Invoker.Type.Chain(maxPriority)` 走指定优先级范围。
- `invokeSpecial()` 和 `newInstanceSpecial()` 用于非虚调用或特殊构造器调用，只在调用链证据明确时使用。
- `deoptimize(Executable)` 用于确认 ART inline 后仍无法命中时的专项处理；先排除 scope、进程、ClassLoader、签名和加载时机问题。
- 现代 `libxposed/api` 不提供资源 Hook。资源修改需求应分解为 Android 资源、View/逻辑观测或运行时方法 Hook。

## Scope 与进程路由

scope 是注入范围，不是最终回调包集合。框架会将模块注入 scope 包声明的常规进程，进程中后续加载的其他包也可能触发回调。因此应使用如下路由顺序：

```text
packageName -> processName -> lifecycle -> ClassLoader -> hook target
```

每个进程独立记录安装状态和 HookHandle。多进程目标要分别取得 ClassLoader、方法签名和调用证据；不要把主进程的类加载器或安装标记复用于 `:remote`、provider 或其他进程。

## Hot Reload 事实

API 102 Hot Reload 只适用于恰好一个 Java 入口类的模块。它不会自动重新调用 `onModuleLoaded()`、`onPackageLoaded()` 或 `onPackageReady()`。旧代码通过 `onHotReloading()` 写入简单、与 ClassLoader 无关的状态；新代码在 `onHotReloaded()` 读取状态和 `getOldHookHandles()`，再选择 `replaceHook()` 或 `unhook()`。

Remote Preferences 是配置共享机制，不等同于 Hot Reload。模块 App 通过 `libxposed/service` 查询运行目标并异步触发热重载；回调可能运行在 Binder 线程，UI 更新需要切回主线程。Native library 是否立即卸载由框架实现决定，Native/JNI 状态需要单独记录和清理。

## APP 逆向到 Hook 的直接路径

对 APK/AAB、DEX、smali、反编译代码、Native so 或运行日志的普通分析，按证据推进即可：

1. 枚举 Manifest、组件、进程、DEX、资源、so、JNI 注册和关键字符串，记录样本版本或哈希。
2. 从入口 Activity、Service、Provider、Receiver、Binder、网络请求或 Native 导出建立候选调用链。
3. 用静态签名和运行日志确认目标类、方法、参数、返回值、ClassLoader、进程和加载时机。
4. 先生成只读日志或参数/返回值观测 Hook，再决定是否修改返回值或调用参数。
5. 用 `XposedModule`、`java_init.list`、`module.prop`、`scope.list` 和 API 102 模板落地模块。
6. 按 `module_loaded -> route -> install_hook -> hook_hit -> fallback` 检查链路；缺失证据标为待验证，同时给出下一条可执行定位命令或最小观测代码。

输出应区分：`已确认事实`、`基于证据的推断`、`待验证假设`、`可直接运行的代码/命令` 和 `验证结果`。
