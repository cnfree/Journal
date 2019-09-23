# HystrixRequestContext跨线程传递对象

## Thread 变量透传
>java中的threadlocal，是绑定在线程上的。你在一个线程中set的值，在另外一个线程是拿不到的。如果在threadlocal的平行线程中，创建了新的子线程，那么这里面的值是无法传递、共享的。这就是透传问题。

常见的变量透传场景：
 * 普通线程的ThreadLocal透传问题
 * sl4j MDC组件中ThreadLocal透传问题
 * Hystrix组件的透传问题

## InheritableThreadLocal 
![ThreadLocal]
* 以上代码在主线程设置了一个简单的threadlocal变量，然后在自线程中想要取出它的值。程序的输出是：**null**
* 解决方法： 将ThreadLocal换成 **InheritableThreadLocal**

## 线程池里的问题
* 当使用线程池时，由于会缓存线程，线程是缓存起来反复使用的。
* 线程池InheritableThreadLocal进行提交，获取的值，有可能是前一个任务执行后留下的，是错误的。
* 只有在任务执行的时候进行传递，才是正确的。

## 解决方案
1. 使用**TransmittableThreadLocal**
![TTL]
2. 使用Spring Hystrix 的 **HystrixRequestContext**

## HystrixRequestContext
* HystrixRequestContext 表示request level的context，用于存储 request level 的数据。与此相对的是 thread level的数据，一般用ThreadLocal来处理
    ```java
    @Test
    public void test() throws InterruptedException {
        // 在当前线程下创建一个HystrixRequestContext对象
        HystrixRequestContext.initializeContext();
        // 创建HystrixRequestVariableDefault作为检索数据的key
        final HystrixRequestVariableDefault<String> variableDefault = new HystrixRequestVariableDefault<>();
        // 将<HystrixRequestVariableDefault,kitty>存储到当前线程的HystrixRequestContext中
        variableDefault.set("kitty");
        // HystrixContextRunnable 是核心, 下面将分析源码:
        HystrixContextRunnable runnable =
                new HystrixContextRunnable(() -> System.out.println(variableDefault.get()));
        // 用子线程执行任务
        new Thread(runnable, "subThread").start();
    }
    ```
  * 以上代码输出结构: kitty。代码并没有显示的将 kitty 从 main线程传递到子线程，也没有利用InheritableThreadLocal  
* 真正存储数据的是HystrixRequestContext，和ThreadLocal一样，存储数据的不是ThreadLocal，而是Thread本身的数据结构 
    ```java
    // 拿到当前线程的存储结构, 用自己作为key, 存储实际的数据。
    public void set(T value) {
        HystrixRequestContext.getContextForCurrentThread().state.put(this, new LazyInitializer<T>(this, value));
    }
    
    public T get() {
        // 拿到当前线程的存储结构, 以自己作为key, 来检索数据
        ConcurrentHashMap<HystrixRequestVariableDefault<?>, LazyInitializer<?>> variableMap = HystrixRequestContext.getContextForCurrentThread().state;
    
        LazyInitializer<?> v = variableMap.get(this);
        if (v != null) {
            return (T) v.get();
        }
        ...
    }
    ```
  ```java
  public class HystrixRequestContext implements Closeable {
      // 利用ThreadLocal, 每个线程各有一份HystrixRequestContext,当然,前提是调用了initializeContext()进行初始化
      private static ThreadLocal<HystrixRequestContext> requestVariables = new ThreadLocal<HystrixRequestContext>();
      
      // 创建一个HystrixRequestContext,并与当前线程关联
      public static HystrixRequestContext initializeContext() {
          HystrixRequestContext state = new HystrixRequestContext();
          requestVariables.set(state);
          return state;
      }
  
      // 获取当前线程关联的HystrixRequestContext, 用的是ThreadLocal
      public static HystrixRequestContext getContextForCurrentThread() {
          HystrixRequestContext context = requestVariables.get();
          if (context != null && context.state != null) {
              return context;
          } else {
              return null;
          }
      }
  
      // 为当前线程设置一个已存在的HystrixRequestContext
      public static void setContextOnCurrentThread(HystrixRequestContext state) {
          requestVariables.set(state);
      }
  
      // 这句单独说 (注意:实际类型不是T,我简化了)
      ConcurrentHashMap<HystrixRequestVariableDefault<?>, T value> state = new ...
  }
  ```
* **ConcurrentHashMap<HystrixRequestVariableDefault<?>, T value> state**
  * 这是实际的存储结构，每个线程关联一个HystrixRequestContext，每个HystrixRequestContext有个Map结构存储数据，key就是HystrixRequestVariableDefault。

## 如何实现request level context? 
  * 实现的秘密就在HystrixContextRunnable和HystrixContextCallable中
    ```java
    // HystrixContextRunnable是个Runnable,一个可用于执行的任务
    public class HystrixContextRunnable implements Runnable {
    
        private final Callable<Void> actual;
        private final HystrixRequestContext parentThreadState;
    
        public HystrixContextRunnable(Runnable actual) {
            this(HystrixPlugins.getInstance().getConcurrencyStrategy(), actual);
        }
        
        public HystrixContextRunnable(HystrixConcurrencyStrategy concurrencyStrategy, final Runnable actual) {
            // 获取当前线程的HystrixRequestContext(如文首的main线程)
            this(concurrencyStrategy, HystrixRequestContext.getContextForCurrentThread(), actual);
        }
    
        // 关键的构造器
        public HystrixContextRunnable(final HystrixConcurrencyStrategy concurrencyStrategy, final HystrixRequestContext hystrixRequestContext, final Runnable actual) {
            
            // 将原始任务Runnable包装成Callable, 创建了一个新的callable
            this.actual = concurrencyStrategy.wrapCallable(new Callable<Void>() {
                @Override
                public Void call() throws Exception {
                    actual.run();
                    return null;
                }
            });
            // 存储当前线程的hystrixRequestContext
            this.parentThreadState = hystrixRequestContext;
        }
    
        @Override
        public void run() {
            // 运行实际的Runnable之前先保存当前线程已有的HystrixRequestContext
            HystrixRequestContext existingState = HystrixRequestContext.getContextForCurrentThread();
            try {
                // 设置当前线程的HystrixRequestContext,来自上一级线程,因此两个线程是同一个HystrixRequestContext
                HystrixRequestContext.setContextOnCurrentThread(parentThreadState);
                try {
                    actual.call();
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            } finally {
                // 还原当前线程的HystrixRequestContext
                HystrixRequestContext.setContextOnCurrentThread(existingState);
            }
        }
    }
    ```
  * Hystrix 的思路是包装Runnable，在执行实际的任务之前，先拿当前线程的HystrixRequestContext初始化实际执行任务的线程的HystrixRequestContext。
  * HystrixContextCallable做的事情和HystrixContextRunnable是一样的，只不过它实现了Callable
  * TransmittableThreadLocal的原理和HystrixRequestContext是完全一样的
  
## HystrixRequestVariableDefault 和ThreadLocal的一些区别  
* 它使用前需要用 HystrixRequestContext.initializeContext() 进行初始化
* 它结束时需使用 HystrixRequestContext.shutdown()进行清理
* 它有一个生命周期方法 shutdown()用来清理资源
* 它会以传引用的方式(线程之间使用的是相同的HystrixRequestVariables)拷贝到下一个线程，主要通过HystrixRequestContext.getContextForCurrentThread()和HystrixRequestContext.setContextOnCurrentThread()两个方法
* 父线程调用shutdown时，子线程的HystrixRequestVariables也会被清理(因为就是一个对象，传的是引用)。
* HystrixRequestContext并不是每个线程都需要的，因此需要根据需要自行进行初始化。

  
  
  
  
  
[ThreadLocal]: img/ThreadLocal.jpeg
[TTL]: img/TTL.webp