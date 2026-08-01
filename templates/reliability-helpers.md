# 程序级重试、超时与降级 Helper 模板

本模板补充 `guides/stability-strategy.md` 的策略说明，提供可复用的 Java Helper 骨架。它适合复杂 Hook、Remote Preferences、动态 ClassLoader、Hot Reload 和 Native 分支初始化等场景。生成真实项目时必须按目标工程替换日志实现、线程模型、错误码和默认值。

## 使用原则

- Hook 回调内不要执行长耗时重试；
- 重试必须有次数上限、间隔上限和错误码；
- 超时失败必须能回退到默认值或 `chain.proceed()`；
- 降级状态必须可观测，不能静默吞异常；
- 不要用无限轮询、主线程 sleep 或全局后台常驻任务掩盖生命周期问题。

## RetryPolicy

用于动态 ClassLoader、service 初始化、Remote Files 读取等可短暂失败的操作。

```java
public final class RetryPolicy {
    public interface Attempt<T> {
        T run() throws Exception;
    }

    public interface Backoff {
        long delayMillis(int attemptIndex, Throwable error);
    }

    private final int maxAttempts;
    private final Backoff backoff;

    public RetryPolicy(int maxAttempts, Backoff backoff) {
        if (maxAttempts < 1) {
            throw new IllegalArgumentException("maxAttempts must be >= 1");
        }
        if (backoff == null) {
            throw new IllegalArgumentException("backoff == null");
        }
        this.maxAttempts = maxAttempts;
        this.backoff = backoff;
    }

    public static RetryPolicy fixedDelay(int maxAttempts, long delayMillis) {
        return new RetryPolicy(maxAttempts, (attempt, error) -> Math.max(0L, delayMillis));
    }

    public <T> T run(String operation, Attempt<T> attempt, T fallback, Logger logger) {
        Throwable lastError = null;
        for (int i = 1; i <= maxAttempts; i++) {
            try {
                T value = attempt.run();
                if (logger != null) {
                    logger.info("event=retry result=ok operation=" + operation + " attempt=" + i);
                }
                return value;
            } catch (Throwable t) {
                lastError = t;
                if (logger != null) {
                    logger.warn("event=retry result=fail operation=" + operation + " attempt=" + i + " code=LSM-RETRY-001", t);
                }
                if (i >= maxAttempts) {
                    break;
                }
                long delay = backoff.delayMillis(i, t);
                if (delay > 0L) {
                    try {
                        Thread.sleep(delay);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        if (logger != null) {
                            logger.warn("event=retry result=abort operation=" + operation + " code=LSM-RETRY-002", e);
                        }
                        return fallback;
                    }
                }
            }
        }
        if (logger != null) {
            logger.warn("event=fallback result=ok operation=" + operation + " recover=default code=LSM-RETRY-003", lastError);
        }
        return fallback;
    }
}
```

使用建议：

- 只在后台线程或明确非主线程路径中使用带 `Thread.sleep()` 的重试；
- Hook 回调中不要 sleep，可改用一次尝试 + 默认值；
- `maxAttempts` 通常为 2 或 3，不要无限重试。

## TimeoutGuard

用于为可阻塞操作提供超时边界。Hook 回调里优先读取已缓存状态；只有确实需要异步任务时才使用。

```java
public final class TimeoutGuard {
    public interface Task<T> {
        T run() throws Exception;
    }

    private final ExecutorService executor;
    private final Logger logger;

    public TimeoutGuard(ExecutorService executor, Logger logger) {
        if (executor == null) {
            throw new IllegalArgumentException("executor == null");
        }
        this.executor = executor;
        this.logger = logger;
    }

    public <T> T run(String operation, long timeoutMillis, Task<T> task, T fallback) {
        Future<T> future = executor.submit(() -> task.run());
        try {
            T value = future.get(timeoutMillis, TimeUnit.MILLISECONDS);
            if (logger != null) {
                logger.info("event=timeout_guard result=ok operation=" + operation + " timeout=" + timeoutMillis);
            }
            return value;
        } catch (TimeoutException e) {
            future.cancel(true);
            if (logger != null) {
                logger.warn("event=timeout_guard result=timeout operation=" + operation + " code=LSM-TIMEOUT-001 recover=default", e);
            }
            return fallback;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            if (logger != null) {
                logger.warn("event=timeout_guard result=abort operation=" + operation + " code=LSM-TIMEOUT-002 recover=default", e);
            }
            return fallback;
        } catch (Throwable t) {
            if (logger != null) {
                logger.warn("event=timeout_guard result=fail operation=" + operation + " code=LSM-TIMEOUT-003 recover=default", t);
            }
            return fallback;
        }
    }
}
```

注意：

- `Future.cancel(true)` 不保证 native、Binder 或不可中断 I/O 立即停止；
- 不要为每次 Hook 命中新建线程池；
- 线程池生命周期必须随模块或进程状态管理，Hot Reload 前应考虑关闭或复用。

## FallbackState

用于记录某个能力是否已降级，避免每次失败都重复执行危险路径。

```java
public final class FallbackState {
    private final AtomicBoolean disabled = new AtomicBoolean(false);
    private volatile String reason = "";

    public boolean isDisabled() {
        return disabled.get();
    }

    public void disable(String reason, Logger logger) {
        if (disabled.compareAndSet(false, true)) {
            this.reason = reason == null ? "unknown" : reason;
            if (logger != null) {
                logger.warn("event=feature_disable result=ok reason=" + this.reason + " code=LSM-FALLBACK-001");
            }
        }
    }

    public String reason() {
        return reason;
    }
}
```

使用示例：

```java
private final FallbackState nativeState = new FallbackState();

private void callNativeIfAvailable(String value) {
    if (nativeState.isDisabled()) {
        logger.info("event=native_call result=skip reason=" + nativeState.reason());
        return;
    }
    try {
        nativeCall(value);
    } catch (UnsatisfiedLinkError e) {
        nativeState.disable("unsatisfied_link", logger);
    } catch (Throwable t) {
        nativeState.disable("native_error", logger);
    }
}
```

## 延迟 ClassLoader 重试 Helper

用于目标类在插件、动态 dex 或后续生命周期才出现的场景。默认只尝试有限次数，并记录 ClassLoader。

```java
public final class DelayedHookInstaller {
    private final AtomicBoolean installed = new AtomicBoolean(false);
    private final RetryPolicy retryPolicy;
    private final Logger logger;

    public DelayedHookInstaller(RetryPolicy retryPolicy, Logger logger) {
        this.retryPolicy = retryPolicy;
        this.logger = logger;
    }

    public boolean install(String className, String methodName, ClassLoader loader, Installer installer) {
        if (installed.get()) {
            if (logger != null) {
                logger.info("event=install_hook result=skip reason=already_installed target=" + className + "#" + methodName);
            }
            return true;
        }

        Boolean ok = retryPolicy.run(
                "install_hook:" + className + "#" + methodName,
                () -> {
                    Class<?> target = loader.loadClass(className);
                    installer.install(target);
                    installed.set(true);
                    return true;
                },
                false,
                logger);

        if (!ok && logger != null) {
            logger.warn("event=install_hook result=skip code=LSM-CL-RETRY-001 target=" + className + "#" + methodName + " loader=" + loader);
        }
        return ok;
    }

    public interface Installer {
        void install(Class<?> targetClass) throws Exception;
    }
}
```

使用规则：

- 适合在已确认的稳定初始化点调用；
- 不要在 Hook 回调里反复创建 `DelayedHookInstaller`；
- 如果仍失败，禁用该 Hook 点并保留模块其他能力。

## Remote Preferences 安全读取

```java
public final class ConfigReader {
    private final TimeoutGuard timeoutGuard;
    private final Logger logger;

    public ConfigReader(TimeoutGuard timeoutGuard, Logger logger) {
        this.timeoutGuard = timeoutGuard;
        this.logger = logger;
    }

    public boolean readBoolean(String key, boolean fallback) {
        if (key == null || key.length() == 0) {
            if (logger != null) {
                logger.warn("event=read_config result=skip code=LSM-CFG-001 reason=empty_key recover=default");
            }
            return fallback;
        }

        Boolean value = timeoutGuard.run(
                "read_config:" + key,
                300L,
                () -> readRemoteBoolean(key, fallback),
                fallback);

        if (value == null) {
            if (logger != null) {
                logger.warn("event=read_config result=fail code=LSM-CFG-002 reason=null_value recover=default key=" + key);
            }
            return fallback;
        }
        return value;
    }

    private Boolean readRemoteBoolean(String key, boolean fallback) {
        // 替换为真实 Remote Preferences 读取逻辑。
        return fallback;
    }
}
```

## Hook 回调中的降级写法

```java
hook(method).intercept(chain -> {
    try {
        Object arg0 = chain.getArg(0);
        if (!(arg0 instanceof String)) {
            logger.warn("event=hook_hit result=skip code=LSM-HOOK-004 recover=proceed reason=bad_arg");
            return chain.proceed();
        }

        boolean enabled = cachedConfig.enabled();
        if (!enabled) {
            return chain.proceed();
        }

        Object original = chain.proceed();
        if (!(original instanceof Boolean)) {
            logger.warn("event=fallback result=ok code=LSM-HOOK-005 recover=original reason=bad_return");
            return original;
        }

        return original;
    } catch (Throwable t) {
        logger.warn("event=fallback result=ok code=LSM-HOOK-006 recover=proceed", t);
        return chain.proceed();
    }
});
```

注意：如果 `chain.proceed()` 本身已执行且后续逻辑抛错，不能再次调用 `chain.proceed()`。真实代码中可用局部变量标记是否已调用：

```java
boolean proceeded = false;
Object original = null;
try {
    original = chain.proceed();
    proceeded = true;
    return original;
} catch (Throwable t) {
    if (proceeded) {
        return original;
    }
    return chain.proceed();
}
```

## Logger 接口占位

模板中的 `Logger` 是占位接口，生成真实代码时应替换为模块内统一日志实现，最终仍应调用 Xposed 日志。

```java
public interface Logger {
    void info(String message);
    void warn(String message);
    void warn(String message, Throwable throwable);
}
```

## 错误码

| 错误码 | 含义 | 回退 |
|---|---|---|
| LSM-RETRY-001 | 单次重试失败 | 继续有限重试 |
| LSM-RETRY-002 | 重试等待被中断 | 返回 fallback |
| LSM-RETRY-003 | 达到最大重试次数 | 返回 fallback |
| LSM-TIMEOUT-001 | 操作超时 | 返回 fallback |
| LSM-TIMEOUT-002 | 操作被中断 | 返回 fallback |
| LSM-TIMEOUT-003 | 操作异常 | 返回 fallback |
| LSM-FALLBACK-001 | 能力被禁用 | 跳过该能力 |
| LSM-CL-RETRY-001 | 延迟 ClassLoader 重试失败 | 禁用该 Hook 点 |

## 验证清单

1. 每个重试都有最大次数；
2. Hook 回调不做 sleep 或长耗时阻塞；
3. 超时后有日志和 fallback；
4. 降级状态能被日志观察；
5. Hot Reload 前后不会留下旧线程池或旧状态；
6. Remote Preferences 读取失败使用默认值；
7. Native 分支失败后 Java 分支仍可运行；
8. `chain.proceed()` 不被错误地重复调用。

## 与其他文件配合

- 稳定性策略：`guides/stability-strategy.md`。
- 防御性错误码：`templates/defensive-error-handling.md`。
- 自动化验证：`guides/validation-checklist.md`。
- 复杂组合：`guides/advanced-combinations.md`。
- 真实故障修复：`cases/failure-fix-walkthroughs.md`。