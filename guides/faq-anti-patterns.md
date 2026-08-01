# FAQ 与常见反模式速查

这个文件把分散在能力范围、案例、迁移和排错分片里的高频问题集中到一处。遇到不确定请求时，先用这里判断方向，再按需读取 `knowledge/` 分片。

## FAQ

### 1. 模块为什么安装后没有效果？

优先检查四件事：

- `module.prop` 是否被打包到 `META-INF/xposed/module.prop`；
- `java_init.list` 是否指向真实入口类；
- `scope.list` 是否包含目标包名；
- 是否重启目标 App、强停目标 App 或重启系统服务。

如果四项都正确，再检查 ClassLoader、进程名、方法签名和 Hook 时机。

### 2. API 102 模块为什么不能把 `libxposed:api` 写成 `implementation`？

`io.github.libxposed:api:102.0.0` 应使用 `compileOnly`。运行时 API 由框架提供，打进 APK 会增加冲突和误判风险。

### 3. 什么时候使用 `libxposed/helper`？

只在复杂查找、混淆定位或需要现代辅助能力时使用。不要默认加入 helper，也不要固定旧版本号；生成前必须以 Maven/Gradle 实际可解析版本为准。

### 4. 什么时候开启 `autoHotReload=true`？

只有模块已经实现清晰的状态清理、Hook 替换和 reload 回调时才开启。默认保持 `false` 更稳妥。

### 5. 旧 `XposedHelpers` 是否还能用？

只用于迁移或兼容场景。现代 API 102 模块应优先使用 `XposedModule` 生命周期、Hooker、HookHandle、Invoker 等现代模型。

### 6. Native Hook 是否适合默认生成？

不适合。Native Hook 需要明确目标 so、符号、ABI、加载时机和回退策略。只有用户明确需要 Native Hook 时才展开。

## 常见反模式

| 反模式 | 风险 | 推荐做法 |
|---|---|---|
| scope 写得过大 | 增加误触发、性能和合规风险 | 只声明目标包名，必要时再扩展 |
| 入口类堆砌 | 生命周期混乱，日志难追踪 | 保留少量入口类，内部按策略分发 |
| 不判断包名和进程 | Hook 到错误进程 | 入口处先判断 package/process |
| 盲 Hook 私有混淆类 | 版本变动后失效 | 优先稳定公开路径，必要时做降级 |
| 方法签名靠猜 | `NoSuchMethod` 或误 Hook | 从源码、反编译或日志确认签名 |
| 缺少异常日志 | 出错后无法定位 | 捕获异常并记录 event/package/process/reason |
| 把 API 依赖打进 APK | 与框架运行时冲突 | `libxposed:api` 使用 `compileOnly` |
| 默认开启 Hot Reload | 旧 Hook 和状态残留 | 默认关闭，确认 reload 安全后再启用 |
| 旧 `XSharedPreferences` 直迁 | 新系统不可用或不稳定 | 使用 `libxposed/service` 与 Remote Preferences |
| Native Hook 不校验 ABI | 加载失败或崩溃 | 检查 ABI、so 名称、符号和加载日志 |

## 逆向分析任务分流

| 用户任务 | 先读取 | 首要产出 |
|---|---|---|
| APK / AAB 静态分析 | `knowledge/01-project-basics.md` + `guides/validation-checklist.md` | 包结构、Manifest、组件、权限、DEX/资源和证据表 |
| 反编译代码 / smali 定位 | `knowledge/01-project-basics.md` + `knowledge/02-hook-api.md` | 类、方法、调用链、参数和版本差异 |
| 动态调试 / Hook | `knowledge/02-hook-api.md` + `guides/troubleshooting-cards.md` | 最小日志 Hook、参数/返回值观测、复现路径 |
| Native / JNI / so 分析 | `knowledge/04-native-migration-helper.md` + `cases/advanced-native-hook.md` | ABI、符号、JNI 注册、加载时机和 Java fallback |
| 协议 / IPC / 行为分析 | `knowledge/03-service-remote-hot-reload.md` + `guides/validation-checklist.md` | 字段、状态机、触发条件和验证样本 |
| SystemUI / system_server | `guides/special-boundaries.md` + `guides/advanced-combinations.md` | 版本证据、最小 scope、恢复方案和影响评估 |

## 快速判断规则

- 信息不足时，先列出包名、进程、版本、类名、方法签名、输入样本和日志缺口；
- 静态定位优先于猜测，动态 Hook 优先从只读日志和最小观测开始；
- 目标涉及复杂生命周期时，优先读取 `knowledge/01-project-basics.md` 和 `knowledge/02-hook-api.md`；
- 目标涉及旧 API 时，先读取 `cases/migration-compat.md`；
- 目标涉及架构质量时，先读取 `cases/real-project-patterns.md`；
- 所有运行时修改都记录 Hook ID、命中条件、跳过原因、异常和回退结果。

## 开发效率工具集

### 快速调试技巧

**1. 实时 Hook 日志查看**

```bash
# 实时过滤模块日志（假设 TAG=LSPosedModule）
adb logcat -s LSPosedModule:* | grep "event="

# 只看 Hook 安装和触发
adb logcat -s LSPosedModule:* | grep -E "(install_hook|hook_hit)"

# 只看错误和警告
adb logcat -s LSPosedModule:W LSPosedModule:E

# 保存完整日志到文件
adb logcat -s LSPosedModule:* > module_debug.log
```

**2. 快速复现 Hook**

使用 `am` 命令快速触发目标 Activity/Service：

```bash
# 启动目标 App 主 Activity
adb shell am start -n com.example.target/.MainActivity

# 发送广播触发 Hook
adb shell am broadcast -a com.example.target.ACTION_TEST

# 启动 Service
adb shell am startservice -n com.example.target/.TargetService

# 强停后重启（触发完整初始化）
adb shell am force-stop com.example.target && \
adb shell am start -n com.example.target/.MainActivity
```

**3. 模块热重载（需要框架支持）**

如果模块开启了 `autoHotReload=true`：

```bash
# 1. 编译并安装新 APK
./gradlew :app:assembleDebug && \
adb install -r app/build/outputs/apk/debug/app-debug.apk

# 2. 触发框架重载（方法因框架而异）
# LSPosed: 在管理界面重新勾选模块
# 或者通过 adb 发送特定广播（需查看框架文档）
```

### APK 快速分析工具链

```bash
# 1. 查看 APK 基本信息
aapt dump badging target.apk | grep -E "(package|launchable-activity|uses-permission)"

# 2. 提取并反编译 DEX
unzip target.apk classes.dex
d2j-dex2jar classes.dex
jd-gui classes-dex2jar.jar

# 3. 查看 Manifest
aapt dump xmltree target.apk AndroidManifest.xml

# 4. 查找目标类和方法（从反编译代码中）
find decompiled-src -name "*.java" | xargs grep -l "targetMethod"

# 5. 快速定位字符串
strings classes.dex | grep "关键字符串"
```

### Hook 代码生成器（命令行）

快速生成基础 Hook 代码框架：

```bash
#!/bin/bash
# generate-hook.sh <package> <class> <method>

PACKAGE=$1
CLASS=$2
METHOD=$3

cat <<EOF
@Override
public void onPackageLoaded(XposedInterface.PackageLoadedParam param) {
    if (!"${PACKAGE}".equals(param.getPackageName())) return;
    if (!param.isFirstPackage()) return;

    try {
        Class<?> clazz = param.getClassLoader().loadClass("${CLASS}");
        var method = clazz.getDeclaredMethod("${METHOD}");
        
        hook(method)
            .setId("${METHOD}_hook")
            .setExceptionMode(XposedInterface.ExceptionMode.DEFAULT)
            .intercept(chain -> {
                log("LSM-HOOK-004 event=hook_hit method=${METHOD}");
                return chain.proceed();
            });
            
        log("LSM-HOOK-001 event=install_hook target=${CLASS}.${METHOD} result=ok");
    } catch (Throwable t) {
        log("LSM-HOOK-003 event=install_hook target=${CLASS}.${METHOD} result=fail", t);
    }
}
EOF
```

使用：
```bash
./generate-hook.sh com.example.target com.example.target.MainActivity onCreate
```

### 性能分析与优化建议

**Hook 性能检查清单**

```text
☐ Hook 数量是否超过 50 个？（建议按需延迟安装）
☐ 是否在热路径上执行反射？（建议缓存 Method/Field）
☐ 是否在 Hook 回调中执行 IO 或网络？（移到后台线程）
☐ 是否记录大量重复日志？（采样或降级）
☐ scope 是否过大？（精确到目标包名）
☐ 是否全局 Hook Activity 生命周期？（按需 Hook 特定类）
```

**内存泄漏检查**

```java
// 避免持有 Activity/Context 引用
private static WeakReference<Context> contextRef;

// 避免静态持有 ClassLoader
// 错误示例：private static ClassLoader cachedLoader;
// 正确示例：使用 WeakHashMap<ClassLoader, ?>

// Hook 回调中避免创建大量临时对象
// 复用 StringBuilder、数组等
```