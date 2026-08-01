# 02 - Hook API 与链式模型

覆盖原知识库第 14-21 章：Hook 基础模型、Chain 规则、优先级、ExceptionMode、HookHandle、Invoker、`hookClassInitializer()`、`deoptimize()`。

## Hook 基础模型

现代 API 使用类似拦截器链的模型。基本写法：

```java
Method method = targetClass.getDeclaredMethod("targetMethod", String.class);

hook(method)
    .setPriority(PRIORITY_DEFAULT)
    .setExceptionMode(ExceptionMode.DEFAULT)
    .intercept(chain -> {
        Object result = chain.proceed();
        return result;
    });
```

核心规则：

- `hook(Executable)` 用于 Hook 方法或构造器；
- 返回 `XposedInterface.HookBuilder`；
- `intercept(...)` 安装 Hook；
- `chain.proceed()` 调用下一个 Hook 或原始方法；
- 不调用 `proceed()` 可以阻断原调用；
- 返回值会成为当前调用结果；
- void 方法和构造器的返回值会被忽略。

## Chain 常用方法

常用 API：

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

重要规则：

- 实例方法的 `getThisObject()` 返回 `this`；
- 静态方法的 `getThisObject()` 返回 `null`；
- `<clinit>` 静态初始化器的 `getThisObject()` 永远为 `null`；
- `getArgs()` 返回不可变列表，不能直接修改；
- 修改参数必须复制参数数组后调用 `chain.proceed(newArgs)`；
- `Chain` 不能缓存、不能跨线程、不能在 `intercept()` 返回后继续使用。

错误示例：

```java
chain.getArgs().set(0, "new");
```

正确示例：

```java
Object[] newArgs = chain.getArgs().toArray();
newArgs[0] = "new";
return chain.proceed(newArgs);
```

## proceed 与 proceedWith

`chain.proceed()` 继续执行后续 Hook 或原始方法。

`chain.proceed(newArgs)` 用于修改参数。

`chain.proceedWith(newThis)` 用于替换 `this` 对象。静态方法 Hook 不应调用 `proceedWith()`。

读取单个参数时要防御 `IndexOutOfBoundsException` 和 `ClassCastException`，不要无保护强转。

## Hook 优先级

常用优先级：

```java
PRIORITY_DEFAULT
PRIORITY_HIGHEST
PRIORITY_LOWEST
```

规则：优先级高的 Hook 先进入；优先级低的 Hook 后进入；返回结果会沿链返回给上游 Hook；多个模块同时 Hook 时，优先级会影响执行顺序。

## ExceptionMode

`ExceptionMode.DEFAULT` 跟随 `module.prop` 中的全局配置。默认 `module.prop` 使用 `exceptionMode=protective`，因此稳定性优先的模块通常使用 `DEFAULT`。

`ExceptionMode.PASSTHROUGH` 适合调试，会让 Hooker 异常继续向上传播，有助于暴露错误，但可能导致目标 App 崩溃。Java API 枚举没有 `PROTECTIVE` 常量；`protective` 是 `module.prop` 的字符串配置值，不是 Java 枚举值。

## HookHandle

`intercept(...)` 返回 `HookHandle`：

```java
HookHandle handle = hook(method).intercept(chain -> chain.proceed());
```

常用能力：

```java
handle.getExecutable();
handle.unhook();
handle.getId();
handle.replaceHook(newHooker);
```

`unhook()` 用于取消 Hook，幂等，可重复调用，常用于清理和热重载。

API 102 支持 Hook id：

```java
hook(method)
    .setId("targetMethodHook")
    .intercept(chain -> chain.proceed());
```

同一模块、同一 Executable、同一 id 的新 Hook 会原子替换旧 Hook，旧 handle 会失效；不同模块之间 id 互相隔离。

热重载中可以用 `replaceHook(...)` 替换旧 Hook。它保留原 executable、优先级、异常模式和 id。Hook 链是快照语义，替换不会影响已经进入的调用。

## Invoker

`Invoker` 用于调用方法或构造器，并绕过访问检查。

方法 Invoker：

```java
Invoker<?, Method> invoker = getInvoker(method);
Object result = invoker.invoke(thisObject, args);
```

构造器 Invoker：

```java
CtorInvoker<?> invoker = getInvoker(constructor);
Object instance = invoker.newInstance(args);
```

`Invoker.Type.ORIGIN` 直接调用原始实现，跳过所有 Hook，适合需要调用 raw method 的场景。

`Invoker.Type.Chain.FULL` 是默认类型，走完整 Hook 链。

`Invoker.Type.Chain(maxPriority)` 从指定优先级位置继续执行 Hook 链，用于高级链控制。

`invokeSpecial()` 用于非虚调用，类似 JNI 的 `CallNonVirtual<type>Method`。`newInstanceSpecial()` 可用父类构造器初始化子类实例，但可能让子类字段未初始化，只适合非常明确的高级场景。

## hookClassInitializer

用于 Hook 静态初始化器 `<clinit>`：

```java
hookClassInitializer(targetClass)
    .intercept(chain -> chain.proceed());
```

规则：

- 如果类已经初始化，Hook 永远不会触发；
- 必须在类初始化前安装 Hook；
- `chain.getThisObject()` 永远为 `null`；
- `chain.getArgs()` 为空；
- `chain.proceed()` 返回 `null`；
- 属于高级场景，普通模块尽量少用。

## deoptimize

```java
boolean ok = deoptimize(executable);
```

用途：当目标方法被 ART inline 导致 Hook 不触发时使用，尤其是 framework 路径可能遇到。

排查顺序必须先于 deoptimize：

1. scope 是否正确；
2. 进程是否正确；
3. ClassLoader 是否正确；
4. 方法签名是否正确；
5. Hook 时机是否太晚；
6. 是否被混淆；
7. 是否 inline；
8. 必要时再考虑 `deoptimize()`。

`deoptimize()` 不是首选方案。优先换 Hook 点；滥用可能影响性能。