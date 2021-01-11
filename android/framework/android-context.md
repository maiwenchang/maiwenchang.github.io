# Context

上下文

Context，ContextWrapper，ContextImpl

Application，Activity，Service

ContentProvider，BroadcastReceiver



```java
public abstract class Context {
    public abstract Looper getMainLooper();
    public abstract boolean bindService(@RequiresPermission Intent service,
            @NonNull ServiceConnection conn, @BindServiceFlags int flags);
  
}
```





