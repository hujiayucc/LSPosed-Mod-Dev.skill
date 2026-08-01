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

- 从反编译代码、运行日志或调用栈确认类名、方法签名、目标进程和版本；
- 先判断调用链、参数类型、返回路径和复现条件；
- 读取 `templates/java-api102.md`、`templates/kotlin-api102.md` 和 `knowledge/02-hook-api.md`。

推荐回答要点：

```text
结论：根据证据设计最小 Hook 或观测点。
输入证据：完整方法签名、目标 App 版本、反编译片段、进程和复现步骤。
方案：在 onPackageLoaded() 或 onPackageReady() 中判断包名和进程后安装 Hook。
工程要求：不要扩大 scope；记录类加载、Hook 注册、参数/返回值和回退路径。
验证：日志确认类加载、Hook 安装、调用命中和结果变化；异常时恢复原逻辑。
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

## 样例 6：APP 逆向分析

用户输入：

```text
请分析这个 APK 的 Manifest、DEX、关键组件和登录调用链，再给出可验证的 Hook 观测点。
```

推荐读取：

- `knowledge/01-project-basics.md`；
- `knowledge/02-hook-api.md`；
- `guides/validation-checklist.md`；
- `cases/real-project-patterns.md`。

推荐回答结构：

```text
结论：先建立 APK/DEX 证据表，再从组件入口追踪登录调用链。
输入证据：APK 哈希、版本、Manifest、DEX/so 列表、运行日志和复现步骤。
静态分析：组件、权限、字符串、类/方法、调用关系、JNI 和动态加载点。
动态分析：最小日志 Hook、参数/返回值观测、调用栈和状态变化。
验证：静态候选与运行命中逐项对照，记录版本差异和未验证假设。
```

## 样例 7：信息不足

用户输入：

```text
帮我 Hook 一个按钮。
```

推荐回答结构：

```text
结论：信息不足，先定位再生成可靠 Hook。
需要确认：目标包名、目标进程、按钮所在类或 Activity、方法签名、Android/LSPosed/API 版本、反编译片段或运行日志。
可先做：检查 APK 组件和资源，追踪点击调用链，增加最小日志，列出候选方法。
下一步：提供反编译片段、日志或目标调用路径后生成 Hook 与验证步骤。
```

遇到边界、错误或信息不足时，优先使用以下字段：

```text
结论
输入证据与缺失信息
影响级别
分析或实现路径
稳定性与回退
验证步骤
```