---
title: java异步编程探究-Future
date: 2019-11-28 15:07:18
tags: 
 - java
 - 异步

categories: 
 - java
 - 异步
---

该文章是Java异步编程探究系列文章的第二章。
如果还没看第一篇文章的话，可以先看下第二篇文章。

上篇文章提到了，异步往往伴随着callback，而且callback多了之后，很容易形成callback hell导致代码被割据到各个callback中。以至于代码难以阅读和维护。

这篇文章谈谈解决callback hell的方式之一，Future。

> 自Java1.8起，终于在jdk中对异步回调有了类库的支持。添加了CompletableFuture。虽然这货添加了，但是可恶的CompletableFuture api极其丑陋。下面细说这个CompletableFuture。



### 鸡肋的Future
> jdk自带的Future没有提供对callback的解决方案，这里讨论Future是为了更好的引出CompletableFuture。

对于Future用法，一般是使用Executor#submit方法，提交一个作业任务并返回一个Future。
```java
ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<String> future = executor.submit(() -> {
            System.out.println("hello world");
            return "executor";
        });
        future.get(); //阻塞到作业任务执行完。
```
这个例子中，向线程池提交一个作业任务并返回一个Future。
但是这个Future没有提供回调的方式，需要你手动调用get方法去获取结果。但是一旦这个Runnable还未执行完的话，那么调用get方法的线程就会被阻塞掉。非常的不方便。
所以这个Future非常不适合线程间的作业交互。也就是说即便有了这个Future，你想要获取结果也很麻烦，因为没结果就阻塞当前线程不是一个好的方法。

下面来分析一下Future的源码，以及它为什么会阻塞。

#### Future api
```java
public interface Future<V> {
  
  //取消当前future代表的任务。
  //mayInterruptIfRunning参数: 是否需要中断线程
  boolean cancel(boolean mayInterruptIfRunning);

  boolean isCancelled();

  boolean isDone();

  //如果任务未执行完，那么将阻塞完到任务执行完。
  //任务执行过程中发现异常的话，异常将会包装到ExecutionException中并抛出。
  V get() throws InterruptedException, ExecutionException;

  //带超时的get方法
  //note；如果timeout时间内未执行完，那么将抛出TimeoutException，而非返回null。因为null也是result的一种。
  V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
接着看看Future的标准实现：FutureTask
FutureTask实现了Runnable接口，所以FutureTask可以当做一个普通的Runnable提交到线程池中。
```java
//代码在AbstractExecutorService类中

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask); //最终也是通过execute方法往线程池提交任务。
        return ftask;
    }
```

```java
public class FutureTask<V> implements RunnableFuture<V> {
 
    private volatile int state; //状态，代表任务执行到哪个阶段， 同时也作为竞态条件
    private static final int NEW          = 0;   //新建阶段
    private static final int COMPLETING   = 1;   //callable已执行完，等待设置结果
    private static final int NORMAL       = 2;   //任务执行完，并设置设置正常的result
    private static final int EXCEPTIONAL  = 3;   //任务执行过程中发生异常
    private static final int CANCELLED    = 4;   //任务被取消
    private static final int INTERRUPTING = 5;   //任务带中断取消的中间状态，也就是执行cannel(true)方法
    private static final int INTERRUPTED  = 6;   //任务带中断取消的最终状态。

    //用户提交的作业任务
    private Callable<V> callable;
    
    //结果，包装正常的result，或非正常的异常结果。异常结果往往throw
    //注意下面的注释，很有意思。很值得研究的一个知识点。这里不展开分析了。
    private Object outcome; // non-volatile, protected by state reads/writes
    
    //执行任务的底层线程，这个对象还充当是否已执行任务的竞态条件。
    private volatile Thread runner;
    
    //调用get方法，get(timeout)方法而阻塞的线程等待节点。
    private volatile WaitNode waiters;

    
    public void run() {
        //这里state和runner两个竟态条件都必须满足
        //如果state不满足，这个future可能被取消或被执行完了
        //如果runner不满足，说明此刻其他线程正在运行着
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) { //上面条件进来时，此时刚好执行了cannel方法修改state条件
                V result;
                boolean ran;
                try {
                    result = c.call();  //执行作业任务
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex); //这个ex可能是callable任务中抛出来的，当然也就包括了InterruptedException
                }
                if (ran)
                    set(result); //设置结果
            }
        } finally {
            runner = null;
            
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s); //避免run方法结束时，存在state == INTERRUPTING 这个中间状态
        }
    }

    
    //设置异常结果
    protected void setException(Throwable t) {
        
        //如果执行过程中，有调用cannel方法的话，那么此刻state状态有可能是CANCELLED,INTERRUPTING,INTERRUPTED中的任意一种。
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) { //如果没修改，那么就设置异常到outcome属性中，最后通过修改state变量发布出去。
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }

    //设置正常结果的方法跟上面异常结果的相似，所以不讲了

    
    //设置完结果后，就开始唤醒阻塞在该future的线程。
    //该方法调用一次有且只有一次。
    private void finishCompletion() {
      for (WaitNode q; (q = waiters) != null;) {
          if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {  //CAS设置watiers，true的话，则竞争成功，逐个唤醒线程。这里有且只有一次能操作成功
              for (;;) {
                  Thread t = q.thread;
                  if (t != null) {
                      q.thread = null;
                      LockSupport.unpark(t); 唤醒线程
                  }
                  WaitNode next = q.next;
                  if (next == null)
                      break;
                  q.next = null; // unlink to help gc
                  q = next;
              }
                break;
         }
      }

      done();   //这个留给后续拓展的。

      callable = null;        // to reduce footprint
    }

  
  //最后来看看get方法， get(timeout)方法类似，分析一个即可
  public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //<=COMPLETING是未完成状态，所以讲当前线程加入等待队列中
        if (s <= COMPLETING) s = awaitDone(false, 0L);
        return report(s); //根据状态输出结果
 } 

 private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {   //被中断，直接throw，通知调用方。
                removeWaiter(q);  //清除当前节点，这里是单向链接的删除操作。
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {  //已完成之后，退出。 加入等待队列后退出当前方法出口也是这里
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) //即将完成，所以使用yield让出时间片的方式
                Thread.yield();
            else if (q == null)
                q = new WaitNode(); //创建等待节点，再下一次循环的时候使用
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);   //竞争waiters条件，成功的话，则入队成功，可以park了（阻塞挂起）
            else if (timed) {  //定时park处理
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }

}
```
上述，就把FutureTask的关键方法梳理了一遍。
根据源码有关state的转向，在注释上也写的很明白。
1. NEW（新建） -> COMPLETING（处理中）-> NORMAL（正常完成）
2. NEW（新建） -> COMPLETING（处理中）-> EXCEPTIONAL（异常完成）
3. NEW（新建） -> CANCELLED（取消）
4. NEW（新建） -> INTERRUPTING（带中断的取消中）-> INTERRUPTED（带中断的取消完成）

所以isCannel方法判断是否是取消的，就判断state>=CANCELLED



### 拓展Future
既然Future#get方法只能阻塞获取节点，而且上面分析了FutureTask也已经留了口子，那么就实现一个基于回调可监听的Future。








### 极其丑陋的CompletableFuture









