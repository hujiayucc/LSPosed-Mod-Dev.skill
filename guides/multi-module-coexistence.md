# 多模块共存场景模板

本文件用于处理同一设备、同一目标 App 或同一进程中存在多个 LSPosed 模块时的设计、排错和审查。重点是避免模块 id、scope、Hook 点、优先级、配置键和日志相互干扰。

## 适用场景

用户提到以下内容时读取本文件：

- 同一目标 App 同时启用了多个模块；
- 两个模块 Hook 同一个类或同一个方法；
- 只有禁用另一个模块后当前模块才生效；
- 多个模块使用相同配置键、日志 tag、native so 名称或 module id；
- scope 交叉导致错误进程加载；
- 需要给模块设置优先级或排查 Hook 顺序。

## 共存基线

每个模块必须独立确认：

```text
module_id=<唯一 id>
module_name=<展示名>
target_package=<目标包名>
target_process=<目标进程或允许列表>
scope=<scope.list 内容>
hook_targets=<类/方法/签名列表>
priority=<是否需要优先级；没有必要则不设置>
config_namespace=<配置命名空间>
log_tag=<唯一日志前缀>
```

默认策略：

1. 不要通过扩大 scope 解决共存问题。
2. 不要假设当前模块一定先执行。
3. 同一 Hook 点必须能在已经被其他模块修改的情况下安全回退。
4. 配置键、日志 tag、native so 名称、Remote Preferences group 要带模块命名空间。
5. 排错时先单模块验证，再启用第二个模块做对照。

## 命名空间模板

建议统一命名：

```text
module_id=com.example.lsposed.feature
log_tag=LSM-Feature
config_prefix=com.example.lsposed.feature.
native_lib=libfeature_lsp.so
hook_id=com.example.lsposed.feature:<class>#<method>(<signature>)
```

配置键示例：

```text
com.example.lsposed.feature.enable_feature
com.example.lsposed.feature.safe_mode
com.example.lsposed.feature.target_process
```

不要使用：

```text
enable
switch
config
hook
native
lsposed
```

这些短键在多模块环境中容易冲突，日志也难以归因。

## scope 交叉排错

| 现象 | 可能原因 | 检查 |
|---|---|---|
| 两个模块都不稳定 | 都 Hook 了同一早期生命周期 | 分别禁用验证，检查 Hook 时机 |
| 当前模块无 `hook_hit` | 另一个模块提前改写返回或中断路径 | 检查目标方法是否仍被调用 |
| 只有主进程生效 | scope 有包名但进程路由漏掉 remote | 记录 package/process 和 route_skip |
| 目标 App 启动崩溃 | 两个模块对同一返回值做不兼容修改 | 使用 `chain.proceed()` 回退并做类型校验 |
| 配置混乱 | 配置键或文件名冲突 | 加模块命名空间和默认值日志 |

最小排查顺序：

1. 只启用当前模块，验证 `module_loaded -> install_hook -> hook_hit`；
2. 只启用另一个模块，记录其 Hook 点和日志；
3. 同时启用，比较 Hook 顺序和返回值变化；
4. 收窄 scope 到最小目标包和目标进程；
5. 给每个 Hook 增加 `hook_id`、`module_id`、`priority`、`process` 日志。

## Hook 优先级使用规则

只有在确实需要顺序时才设置优先级。多数模块应依赖稳定 Hook 点和安全回退，而不是依赖全局顺序。

应先确认：

```text
why_priority_needed=<为什么需要顺序>
other_module=<另一个模块名称或 id>
same_method=<是否 Hook 同一方法>
expected_order=<before|after|no_requirement>
fallback_if_order_changed=<顺序变化时如何回退>
```

建议：

- 只是记录日志，不需要优先级；
- 只是读取参数，一般不需要优先级；
- 修改返回值或跳过原方法时，必须考虑其他模块已修改结果；
- 顺序要求无法保证时，输出风险并建议拆分 Hook 点或改为只读观测。

## API 102 Hook 片段

示例目标：同一方法可能被其他模块 Hook，本模块只在参数合法时修改结果，否则保留链路。

```java
private static final String MODULE_ID = "com.example.lsposed.feature";
private static final String TAG = "LSM-Feature";
private static final String TARGET_PACKAGE = "com.example.app";

private void routeAndInstallFeatureHook(XposedModuleInterface.PackageReadyParam param) {
    if (!TARGET_PACKAGE.equals(param.getPackageName())) {
        log(Log.INFO, TAG, "event=route_skip module=" + MODULE_ID + " reason=package_mismatch");
        return;
    }
    if (!param.isFirstPackage()) {
        log(Log.INFO, TAG, "event=route_skip module=" + MODULE_ID + " reason=not_first_package package=" + param.getPackageName());
        return;
    }
    installFeatureHook(param.getClassLoader());
}

private void installFeatureHook(ClassLoader classLoader) {
    try {
        Class<?> targetClass = classLoader.loadClass("com.example.app.FeatureManager");
        Method method = targetClass.getDeclaredMethod("isEnabled", String.class);
        String hookId = MODULE_ID + ":FeatureManager#isEnabled(String)";

        hook(method).intercept(chain -> {
            Object arg0 = chain.getArg(0);
            if (!(arg0 instanceof String)) {
                log(Log.WARN, TAG, "event=hook_hit result=skip hook_id=" + hookId + " reason=bad_arg");
                return chain.proceed();
            }

            Object original = chain.proceed();
            if (!(original instanceof Boolean)) {
                log(Log.WARN, TAG, "event=fallback result=ok hook_id=" + hookId + " recover=proceed reason=bad_return");
                return original;
            }

            boolean originalValue = (Boolean) original;
            boolean enabled = shouldEnableFeature((String) arg0, originalValue);
            log(Log.INFO, TAG, "event=hook_hit result=ok hook_id=" + hookId + " original=" + originalValue + " final=" + enabled);
            return enabled;
        });

        log(Log.INFO, TAG, "event=install_hook result=ok module=" + MODULE_ID + " hook_id=" + hookId);
    } catch (ClassNotFoundException e) {
        log(Log.WARN, TAG, "event=install_hook result=skip module=" + MODULE_ID + " code=LSM-CL-001", e);
    } catch (NoSuchMethodException e) {
        log(Log.WARN, TAG, "event=install_hook result=skip module=" + MODULE_ID + " code=LSM-SIG-001", e);
    } catch (Throwable t) {
        log(Log.ERROR, TAG, "event=install_hook result=fail module=" + MODULE_ID + " code=LSM-HOOK-001", t);
    }
}
```

要点：

- `chain.proceed()` 后再判断原始返回，避免不必要地打断其他模块；
- 日志包含 `module_id` 和 `hook_id`；
- 参数或返回值不符合预期时回退，不抛异常；
- 不依赖另一个模块的内部实现。

## 多模块共存审查清单

| 检查项 | 通过条件 |
|---|---|
| module id | 唯一且稳定 |
| scope | 只包含必要目标包 |
| 进程路由 | 每个目标进程明确记录 |
| Hook 点 | 同一方法冲突已说明 |
| Hook 顺序 | 不依赖顺序，或已说明优先级原因 |
| 配置键 | 带模块命名空间 |
| 日志 tag | 能区分模块和 Hook 点 |
| 回退 | 其他模块改变返回值时仍能安全处理 |
| 验证 | 单模块和多模块两组日志均可比较 |

## 用户输入模板

```text
同一目标 App 上有多个 LSPosed 模块共存。当前模块 id 是 ...，另一个模块是 ...。
目标包名 ...，目标进程 ...，两个模块是否 Hook 同一方法：是/否。
这是 module.prop、scope.list、日志和两个模块的 Hook 点，请帮我判断冲突、优先级和最小修复方案。
```

## 回答模板

```text
结论：冲突明确 / 可能冲突 / 信息不足。
证据：scope 交叉、进程交叉、同一 Hook 点、日志顺序或返回值变化。
最小修复：命名空间隔离、scope 收窄、Hook 点拆分、参数/返回值 Guard、必要时设置优先级。
验证：单独启用 A、单独启用 B、同时启用 A+B，对比 module_loaded/install_hook/hook_hit/fallback 日志。
风险：不要通过扩大 scope、跳过原方法或依赖未验证顺序来掩盖冲突。
```

## 与其他文件配合

- 多进程组合：`guides/advanced-combinations.md`。
- 模块不生效：`guides/troubleshooting-cards.md`。
- 修复前后对比：`cases/failure-fix-walkthroughs.md`。
- 防御性代码：`templates/defensive-error-handling.md`。
- 验证门禁：`guides/validation-checklist.md`。