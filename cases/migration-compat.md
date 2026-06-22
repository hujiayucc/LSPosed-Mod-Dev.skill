# 旧 API 迁移与兼容案例索引

本文件用于在旧模块迁移任务中低 token 加载。完整细节见 `LSPosed-Mod-Dev.full.knowledge.md` 的旧 API 兼容、迁移和真实案例章节。

## 1. 迁移原则

默认新模块不要使用旧 API。只有以下场景才考虑兼容：

- 用户已有旧 Xposed 模块；
- 需要逐步迁移到现代 API；
- 需要同时兼容旧 LSPosed API 100/101；
- 旧 Hook 逻辑稳定，短期只替换入口和元数据。

## 2. 旧入口到现代入口

旧入口：

```text
assets/xposed_init
IXposedHookLoadPackage
handleLoadPackage(...)
```

现代入口：

```text
META-INF/xposed/java_init.list
XposedModule
onModuleLoaded(...)
onPackageLoaded(...)
onPackageReady(...)
```

迁移策略：

1. 先新增 `module.prop`；
2. 新增 `java_init.list`；
3. 入口类继承 `XposedModule`；
4. 将旧 `handleLoadPackage` 分发逻辑迁到 `onPackageLoaded` 或 `onPackageReady`；
5. 保留旧 Hook 逻辑但包裹日志和异常处理；
6. 再逐步把 `XposedHelpers` 替换为现代 `hook(method).intercept(...)`。

## 3. XposedHelpers 迁移

旧写法：

```java
XposedHelpers.findAndHookMethod(className, classLoader, methodName, ...)
```

现代思路：

```java
Class<?> target = classLoader.loadClass(className);
Method method = target.getDeclaredMethod(methodName, ParamType.class);
hook(method).intercept(chain -> chain.proceed());
```

注意：

- 现代 API 更强调类型明确；
- 方法签名必须自己解析；
- 找不到类/方法时应记录并跳过；
- 旧 `XC_MethodHook` 的 before/after 逻辑需要改写为 `Chain` 模式。

## 4. XSharedPreferences 迁移

旧思路：跨进程读模块 SharedPreferences。

现代思路：

- 使用 `libxposed/service`；
- App 侧通过 `XposedService` 写 `RemotePreferences`；
- Hook 侧读取远程配置；
- Hook 侧通常只读，不负责写；
- 大内容用 Remote Files。

## 5. API 100/101 兼容经验

可借鉴 `MiHealth_AmapFix` 的兼容入口思路：

- 同时支持构造器风格和 `onModuleLoaded()`；
- API 101+ 等待 `onPackageReady()`；
- `getClassLoader()` 失败时尝试 `getDefaultClassLoader()`；
- 用 `process|package|classLoader` 防重复安装；
- 旧 API Hook 逻辑只作为过渡。

## 6. 迁移风险

常见风险：

- 同时保留 legacy 和 modern 入口导致重复 Hook；
- 旧 API 被打包或依赖冲突；
- `assets/xposed_init` 和 `java_init.list` 指向不同入口；
- scope 规则和旧模块行为不一致；
- `onPackageLoaded()` 时机过早；
- 旧 `XSharedPreferences` 在新系统不可用。

## 7. 推荐回答方式

当用户要求迁移旧模块时，输出：

```text
迁移结论
旧模块入口识别
目标 API 版本
module.prop / java_init.list / scope.list
入口类迁移代码
旧 Hook 逻辑保留策略
逐步替换 XposedHelpers 的计划
验证步骤
回滚方案
```