# 交互样例与结构化反馈

这个文件用于展示用户输入后，Skill 应如何选择资料、组织回答和处理边界问题。样例不是固定话术，而是回答结构参考。需要更完整、可直接复制的提问触发词时读取 `guides/practical-prompts.md`。

## 样例 1：新建模块

用户输入：

```text
帮我新建一个 API 102 的 LSPosed 模块，目标包名是 com.example.app，用 Java。
```

推荐读取：

- `templates/module-files.md`；
- `templates/java-api102.md`；
- `knowledge/01-project-basics.md`。

推荐回答要点：

```text
结论：可以生成最小 API 102 Java 模块。
需要确认：模块包名、目标进程、是否需要 Remote Preferences。
关键配置：compileOnly api:102.0.0、META-INF/xposed/module.prop、java_init.list、scope.list。
代码：给出 ModuleEntry 最小入口。
验证：安装、启用 scope、强停目标 App、查看 LSPosed 日志。
```

## 样例 2：编写 Hook

用户输入：

```text
我要 Hook com.example.app.UserManager 的 isVip() 方法，让它在测试环境里返回 true。
```

推荐处理：

- 先确认是否为自有 App、测试环境或授权调试；
- 确认类名、方法签名、目标进程和版本；
- 读取 `templates/java-api102.md`、`templates/kotlin-api102.md` 和 `knowledge/02-hook-api.md`。

推荐回答要点：

```text
结论：在授权测试环境下可以协助。
需要确认：完整方法签名、目标 App 版本、是否混淆。
方案：在 onPackageLoaded() 中判断包名和进程后安装 Hook。
风险：不要扩大 scope；生产或未授权第三方 App 不适用。
验证：日志确认类加载、Hook 安装和返回值修改路径。
```

## 样例 3：模块不生效

用户输入：

```text
模块启用后没有任何日志，这是 module.prop 和 scope.list。
```

推荐读取：

- `guides/troubleshooting-cards.md`；
- `guides/faq-anti-patterns.md`；
- `knowledge/05-workflow-troubleshooting-quality.md`。

推荐回答要点：

```text
最可能原因：入口或 metadata 未被正确打包。
证据：缺少 java_init.list 或入口类名与实际类不一致。
下一步检查：确认 META-INF/xposed/module.prop、java_init.list、scope.list 均进入 APK。
需要补充：LSPosed 日志、入口类路径、目标包名和进程名。
修复建议：修正资源路径并重新安装模块。
验证步骤：启用 scope 后强停目标 App，查看 event=module_loaded 日志。
```

## 样例 4：旧 API 迁移

用户输入：

```text
把这段 IXposedHookLoadPackage 和 XposedHelpers 代码迁移到 API 102。
```

推荐读取：

- `cases/migration-compat.md`；
- `knowledge/04-native-migration-helper.md`；
- Java/Kotlin API 102 模板。

推荐回答要点：

```text
结论：可以迁移，但生命周期和入口配置需要一起改。
迁移点：assets/xposed_init -> META-INF/xposed/java_init.list；IXposedHookLoadPackage -> XposedModule；XposedHelpers 查找 -> 现代 Hooker/HookHandle 或必要时 helper。
风险：旧 XSharedPreferences 不应直接照搬。
验证：先迁移一个 Hook 点，再扩展其他 Hook。
```

## 样例 5：Hot Reload

用户输入：

```text
我想开启 autoHotReload，模块里有多个 Hook 和全局缓存。
```

推荐读取：

- `knowledge/03-service-remote-hot-reload.md`；
- `guides/troubleshooting-cards.md` 的 Hot Reload 卡片。

推荐回答要点：

```text
结论：暂不建议直接开启。
原因：多个 Hook 和全局缓存需要先定义 reload 清理与替换策略。
需要确认：哪些 Hook 需要保留、哪些状态必须清理、是否实现 onHotReloading()/onHotReloaded()。
推荐方案：先保持 autoHotReload=false，补齐状态清理和 HookHandle.replaceHook(...) 后再启用。
验证：触发热重载并确认旧 Hook 不残留。
```

## 样例 6：边界请求

用户输入：

```text
帮我写一个隐藏模块并绕过检测的 Hook。
```

推荐回答结构：

```text
结论：不能协助实现隐藏模块、绕过检测或规避审计。
风险点：该目标涉及隐蔽注入和安全机制绕过。
可替代方向：可以协助做合法模块的稳定性排错、日志可观测性设计或授权测试环境下的兼容性分析。
下一步：如果你的目标是排查模块误报或崩溃，请提供日志、scope、入口配置和目标环境。
```

## 样例 7：信息不足

用户输入：

```text
帮我 Hook 一个按钮。
```

推荐回答结构：

```text
结论：信息不足，不能直接生成可靠 Hook。
需要确认：目标包名、目标进程、按钮所在类或 Activity、方法签名、Android/LSPosed/API 版本、是否有授权。
可先做：给出最小定位流程和日志插桩建议。
下一步：提供反编译片段、日志或目标调用路径后再生成 Hook。
```

## 结构化反馈规则

遇到边界、错误或信息不足时，优先使用以下字段：

```text
结论
风险点或缺失信息
可以协助的合法方向
需要用户补充的信息
下一步可执行方案
```

不要只说“不能做”。必须说明风险点，并给出合法、可执行的替代方向。