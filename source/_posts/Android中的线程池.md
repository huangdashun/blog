---
title: Android中的线程池
date: 2017-10-31 19:54:04
tags:
---
# 1.简述线程池的优点
1. 重用线程池中的线程,避免因为频繁创建线程和销毁线程造成的性能开销.
2. 能有效的控制线程池中的最大并发数,避免大量的线程之间抢占cpu的资源造成的阻塞.
3. 能够对线程简单的管理,并提供定时任务和指定间隔时间执行的功能.
# 2.ThreadPoolExecutor
ThreadPoolExecutor是线程池的真正实现.首先观察其构造函数.

```
/**
 * Created by hs on 2017/10/31.
 */
public class ThreadPoolExecutor {
    //线程池的核心线程数,除非将allowCoreThreadTimeOut设置为true(则由keepAliveTime指定超时时间,超时会被终止)
    //否则核心线程会即时闲置也不会终止
    int corePoolSize;
    //最大线程数,活动线程数超过该数量,后续会排队
    int maximumPoolSize;
    //超时时长,作用于非核心线程和将allowCoreThreadTimeOut设置为true的核心线程
    long keepAliveTime;
    //时间单位,用于指定keepAliveTime,单位有TimeUnit.MILLISECONDS,TimeUnit.SECONDS等
    TimeUnit unit;
    //线程池的任务队列,通过线程池的execute方法提交的Runnable会储存该队列
    BlockingQueue<Runnable> workQueue;
    //线程工厂接口,只有一个方法Thread new Thread(Runnable r),可以为线程池创建新线程
    ThreadFactory threadFactory;
}
```
ThreadPoolExcutor执行任务时的规则(假设当前线程池中的线程数量为*currentSize*):

1. 当*currentSize<corePoolSize*,启动一个核心线程来执行任务.
2. 当*currentSize>=corePoolSize*,并且*workQueue未满*,会被添加到workQueue中等待执行.
3. 当*workQueue已满*,并且*currentSize<maximumPoolSize*,会启动一个非核心线程来执行任务.
4. 如果*currentSize>= maximumPoolSize,workQueue*已满,这时ThreadPoolExecutor会拒绝执行并调用::RejectedExecutionHandler::的rejectExecution方法来通知调用者.
上面提到一个RejectedExecutionHandler,其实ThreadPoolExecutro还有一个参数,就是它,它是负责抛出处理异常的,可选值:CallerRunsPolicy,AbortPolicy(默认),DiscardPolicy和DiscardOldestPolicy.

# 线程池的分类
## FixedThreadPool
描述:它是一种线程数量固定的线程池,且只有核心线程,除非线程池关闭,就算空闲也不会回收.
优点:能够更加快速的响应外界的请求.

```
public static ExecutorService newFixedThreadPool(int nThreads) { return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
    }
代码使用:
    Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("我是newFixedThreadPool");
            }
        };
        Executors.newFixedThreadPool(4).execute(task);
```
## CachedThreadPool
描述:它是一种线程数量不定的线程池,只有非核心线程,最大线程数为Integer.Max_Value,即无限大.如果线程池中的线程都处于活动状态,会创建新的线程,否则会利用空闲线程处理任务.但是有超时机制60秒,超过会被回收.
特点:适合处理大量的耗时较少的任务.

```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
代码使用:
 Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("我是newCachedThreadPool");
            }
        };
        Executors.newCachedThreadPool().execute(task);
```
## ScheduledThreadpool
描述:核心线程数量是固定的,而非核心线程数是没有限制的,而且非核心线程闲置会立即回收.
特点:用于执行定时任务和具有固定周期的重复任务.

```
 public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
 public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
代码使用:
    Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("我是newScheduledThreadPool");
            }
        };
//延迟1000毫秒后执行task
        Executors.newScheduledThreadPool(3).schedule(task,1000,TimeUnit.MILLISECONDS);
//延迟100毫秒后,每隔1000毫秒执行一次task   Executors.newScheduledThreadPool(3).scheduleAtFixedRate(task,100,1000,TimeUnit.MILLISECONDS);
```
## SingleThreadExecutor
描述:线程池内部只有一个核心线程,它确保所有的任务都在同一个线程中按照顺序执行.
特点:使得任务之间不需要处理线程同步的问题.

```
   public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
代码使用:
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("我是newSingleThreadExecutor");
            }
        };
        Executors.newSingleThreadExecutor().execute(task);
```
---
整理了一遍线程池的概念和使用,多谢Android开发艺术探究给我指明了方向,感谢[Android中常见的4种线程池](http://blog.csdn.net/seu_calvin/article/details/52415337),该博文的例子真的是生动形象,大家可以阅读一下.