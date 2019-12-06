---
title: java异步编程探究-Future
date: 2019-11-28 15:07:18
tags: 
 - java
 - 异步
 - CompletionStage
 - CompletableFuture
 - ListenableFuture
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
  //任务执行过程中发生异常的话，异常将会包装到ExecutionException中并抛出。
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
    private static final int COMPLETING   = 1;   //callable已执行完，下一步马上设置结果
    private static final int NORMAL       = 2;   //任务执行完，并设置设置正常的result
    private static final int EXCEPTIONAL  = 3;   //任务执行过程中发生异常
    private static final int CANCELLED    = 4;   //任务被取消
    private static final int INTERRUPTING = 5;   //任务带中断取消的中间状态，也就是执行cannel(true)方法
    private static final int INTERRUPTED  = 6;   //任务带中断取消的最终状态。

    //用户提交的作业任务，提交的作业任务是Runnable的话，最终也是包装成Callable的。
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
            if (c != null && state == NEW) { //还没有线程执行过，且没有cancel。
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
		//这种情况是try里面的if条件没满足。即被cancel了。避免run方法结束时，存在state == INTERRUPTING 这个中间状态
                handlePossibleCancellationInterrupt(s);
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
                      LockSupport.unpark(t); //唤醒线程
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
具体源码在[ListenableFuture](https://github.com/wang007/pandora/tree/master/pandora-ext/src/main/java/com/github/pandora/listenable)
```java
        //实现后的效果
        ListenableExecutorService executor = ListenableExecutor.create(Executors.newSingleThreadExecutor());
        executor.submit(() -> {
            //do something
            return "submit";
        }).addHandler(ar -> {
            if (ar.succeeded()) {
                System.out.println("submit result -> " + ar.result());
            } else {
                ar.cause().printStackTrace();
            }
        });

        //或者用链式串联异步结果
        executor.submit(() -> {
            //do something
            return "submit";
        }).flatMap(str -> {
            System.out.println("result");
            return "other";
        }).map(str -> {
            //do something
            return str.length();
        }).addHandler(ar -> {
            if (ar.succeeded()) {
                System.out.println("result length -> " + ar.result());
            } else {
                ar.cause().printStackTrace();
            }
        });
```

### 极其丑陋的CompletableFuture
> CompletableFuture继承了CompletionStage,java中的阻塞Future，所以CompletableFuture本身还是可以阻塞的。

#### CompletionStage
CompletionStage所有的方法，定义如何获取结果并做响应的操作。但是没有定义如何为这个异步结果设置结果，而设置这个操作直接就实现在具体的实现类CompletableFuture中，这就显得非常糟糕了，使用者必须去依赖CompletableFuture这个具体的实现了。
理想情况下，应该设置一个专门设置异步结果的接口。假设叫做Promise。 Promise只需要关注如何写，CompletionStage只需要关注如何获取结果。
而Netty就是这么做的。参考Netty的Promise，Future。

CompletionStage api的设计同样非常糟糕，重复功能相同的api。以至于后面我用同一个实现方法，其他接口方法都可以直接复用这个实现方法。
而且接口方法很难理解，远没有reactive的命名简洁。

下面就先给api做一个分类。后面使用的时候可以参考这个分类使用。
#### 1. 在异步结果正常完成时调用。区别在于入参和出参。 fuck api  相当于reactive#map
     1. {@link #thenApply(Function),#thenApplyAsync(Function),#thenApplyAsync(Function, Executor)}
     2. {@link #thenAccept(Consumer),#thenAcceptAsync(Consumer),#thenAcceptAsync(Consumer, Executor)}
     3. {@link #thenRun(Runnable),#thenRunAsync(Runnable),#thenAcceptAsync(Consumer, Executor)}
#### 2. 当两个异步结果都正常完成时调用，区别在于入参和出参。fuck api  相当于reactive#zipWith
     1. {@link #thenCombine(CompletionStage, BiFunction),#thenCombineAsync(CompletionStage, BiFunction)}
        {@link #thenCombineAsync(CompletionStage, BiFunction, Executor)}
     2. {@link #thenAcceptBoth(CompletionStage, BiConsumer),#thenAcceptBothAsync(CompletionStage, BiConsumer),
        {@link #thenAcceptBothAsync(CompletionStage, BiConsumer, Executor)}
     3. {@link #runAfterBoth(CompletionStage, Runnable),#runAfterBothAsync(CompletionStage, Runnable)}
        {@link #runAfterBothAsync(CompletionStage, Runnable, Executor)}
#### 3. 两个异步结果任意其中之一正常完成时调用，区别在于入参和出参。 fuck api  相当于reactive#ambWith && map
     1. {@link #applyToEither(CompletionStage, Function),#applyToEitherAsync(CompletionStage, Function)}
        {@link #applyToEitherAsync(CompletionStage, Function, Executor)}
     2. {@link #acceptEither(CompletionStage, Consumer),#acceptEitherAsync(CompletionStage, Consumer)}
        {@link #acceptEitherAsync(CompletionStage, Consumer, Executor)}
     3. {@link #runAfterEither(CompletionStage, Runnable),#runAfterEitherAsync(CompletionStage, Runnable)}
        {@link #runAfterEitherAsync(CompletionStage, Runnable, Executor)}
#### 4. 当前一个异步结果正常完成时产生一个新的不同类型的异步结果。  挺好。 相当于reactive#flatMap        
    1. {@link #thenCompose(Function),#thenComposeAsync(Function, Executor),#thenComposeAsync(Function)}
#### 5. 当异步结果异常完成时调用  相当于reactive#onErrorReturn
    1. {@link #exceptionally(Function)}
#### 6. 当异步结果完成(包括正常或异常)时调用。 相当于reactive#doOnSubscribe
    1. {@link #whenComplete(BiConsumer),#whenCompleteAsync(BiConsumer),#whenCompleteAsync(BiConsumer, Executor)}
    2. {@link #handle(BiFunction),#handleAsync(BiFunction),#handleAsync(BiFunction, Executor)}

所以一共就6种功能的方法，用reactiveX命名操作符概括的话就是map, zipWith, ambWith(or) && map, flatMap, onErrorReturn, doOnSubscribe.
async结尾的方法，是带切换线程的方法。

#### CompletionStage的实现
> CompletableFuture也并没有实现一个功能相同的方法，其他功能相同的方法复用。只是区别有没有线程池(async结尾的方法)做一个复用。
  由于源码实现比较多，所以我相同功能的方法拿出一个来分析即可，其他基本都一样。
  也就是说：thenApply和thenApplyAsync做一个实现。thenAccept和thenAcceptAsync又做一个实现分类，等等。
  
1. map操作符 == thenApply方法
```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
        
        //用于包装异常结果，取结果时，result instanceOf AltResult 即可判断是否是异常结果
        static final class AltResult {
            final Throwable ex; 
            AltResult(Throwable ex) { this.ex = ex; }
        }
        static final AltResult NIL = new AltResult(null);  //当ex == null, 代表成功的null结果。
    
        volatile Object result;    // 
        volatile WaitNode waiters; // get，get(timeOut)方法调用阻塞的线程节点
        volatile CompletionNode completions;  //操作符对应的操作符节点。结果未完成时需要保存下来，待结果完成时在回调操作符。
        
        //操作符节点是用单向链表保存起来的
        static final class CompletionNode {
            final Completion completion;
            volatile CompletionNode next;
            CompletionNode(Completion completion) { this.completion = completion; }
       }
       
       public <U> CompletableFuture<U> thenApply
               (Function<? super T,? extends U> fn) {
               return doThenApply(fn, null);
       }
       
       //最后调用到doThenApply方法
       //fn：操作符对应的操作，一般以lambda的形式提供，
       //e：是否需要切换线程，async结尾的方法 e != null
       private <U> CompletableFuture<U> doThenApply
        (Function<? super T,? extends U> fn, Executor e) {
        if (fn == null) throw new NullPointerException();
        
        //创建新的异步结果并在该方法结束后返回这个新的异步结果。
        // 在当前异步结果完成时，设置这个新的异步结果，那么就像管道一样，result在这CompletableFuture组成的管道上流通。
        CompletableFuture<U> dst = new CompletableFuture<U>();  
        ThenApply<T,U> d = null;
        Object r;
        //结果未完成时，尝试加入到操作符单向链表上
        if ((r = result) == null) {
            CompletionNode p = new CompletionNode
                (d = new ThenApply<T,U>(this, fn, dst, e));
            while ((r = result) == null) {
                //将操作符节点添加到链表头节点上，加入成功后，退出循环。
                if (UNSAFE.compareAndSwapObject
                    (this, COMPLETIONS, p.next = completions, p))
                    break;
            }
        }
        //此时，有两种情况 1. 结果已完成，2.操作符节点以成功加入链表中
        //加入if的临界条件：结果已完成，且操作符未执行过
        if (r != null && (d == null || d.compareAndSet(0, 1))) {
            T t; Throwable ex;
            if (r instanceof AltResult) { //result是异常结果或者null。
                ex = ((AltResult)r).ex; //ex == null时，null结果的完成结果。 ex != null时，异常结果。
                t = null;
            }
            else { //result是正常结果且非null。
                ex = null;
                @SuppressWarnings("unchecked") T tr = (T) r;
                t = tr;
            }
            U u = null;
            if (ex == null) { //ex == null，正常结果。另外：异常结果时，不执行操作符.
                try {
                    if (e != null) //入参线程池 != null，操作符在线程池中执行
                        execAsync(e, new AsyncApply<T,U>(t, fn, dst));
                    else
                        u = fn.apply(t); //执行操作符的操作。
                } catch (Throwable rex) {
                    ex = rex;
                }
            }
             //e == null说明即再次执行操作符操作。若e != null，说明操作符在线程池e中完成，那么这个新的异步结果的结果也会在线程池e中完成
             //ex != null，异常结果，则不需要借用线程池执行操作符操作了
            if (e == null || ex != null)
                //最终把结果设置到新的异步结果上
                dst.internalComplete(u, ex); 
        }
        helpPostComplete();   //这个方法非常糟糕，会让操作符的回调执行所在的线程模糊。后续高版本的jdk8好像重写CompletableFuture，把它给去掉了
        return dst;
    }
    //ThenApply对象，挂载到操作符链表上的操作跟if块的操作差不多，这里就不说了
    
    //再看看提交到线程池e中的AsyncApply如何操作的
    static final class AsyncApply<T,U> extends Async {
            final T arg;
            final Function<? super T,? extends U> fn;
            final CompletableFuture<U> dst;
            AsyncApply(T arg, Function<? super T,? extends U> fn,
                       CompletableFuture<U> dst) {
                this.arg = arg; this.fn = fn; this.dst = dst;
            }
            public final boolean exec() {
                //还是跟if块的代码差不多。
                //这个d，就是doThenApply方法传进来的dst。
                CompletableFuture<U> d; U u; Throwable ex;
                if ((d = this.dst) != null && d.result == null) {
                    try {
                        u = fn.apply(arg);
                        ex = null;
                    } catch (Throwable rex) {
                        ex = rex;
                        u = null;
                    }
                    d.internalComplete(u, ex);
                }
                return true;
            }
            private static final long serialVersionUID = 5232453952276885070L;
        }
        
        //为异步结果设置结果，典型的CAS应用
        final void internalComplete(T v, Throwable ex) {
                if (result == null)
                    UNSAFE.compareAndSwapObject
                        (this, RESULT, null,
                         (ex == null) ? (v == null) ? NIL : v :
                         new AltResult((ex instanceof CompletionException) ? ex :
                                       new CompletionException(ex)));
                postComplete(); // help out even if not triggered，非常糟糕，在这里调用帮助处理
        }
        
        final void postComplete() {
                WaitNode q; Thread t;
                //唤醒阻塞的线程节点
                while ((q = waiters) != null) {
                    if (UNSAFE.compareAndSwapObject(this, WAITERS, q, q.next) &&
                        (t = q.thread) != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                }
                
                //调用操作符链表上所有操作符节点，c 就是ThenApply的父类
                CompletionNode h; Completion c;
                while ((h = completions) != null) {
                    if (UNSAFE.compareAndSwapObject(this, COMPLETIONS, h, h.next) &&
                        (c = h.completion) != null)
                        c.run();
                }
        }
        
        //设置正常结果
         public boolean complete(T value) {
                 //如果 value == null, result 设置成 NIL.
                 boolean triggered = result == null &&
                      UNSAFE.compareAndSwapObject(this, RESULT, null,
                           value == null ? NIL : value);
                      postComplete();  //处理阻塞等待的线程节点和操作符，这里处理就没毛病
                      return triggered;
         }
         
         //设置异常结果
         public boolean completeExceptionally(Throwable ex) {
                 if (ex == null) throw new NullPointerException();
                 boolean triggered = result == null &&
                     UNSAFE.compareAndSwapObject(this, RESULT, null, new AltResult(ex));  //异常结果用AltResult包装
                 postComplete();  //处理阻塞等待的线程节点和操作符，这里处理就没毛病
                 return triggered;
          }
         
}
```
为什么说helpPostComplete(), internalComplete方法中postComplete方法很糟糕呢？
这会让操作符在哪个线程处理变得很迷，如果对一些线程敏感的代码，那么可能导致意想不到的bug，而且不了解这一机制的话，这bug也很难会被发现。

举个例子
```java
    public static void main(String[] args) throws InterruptedException {
        CompletableFuture<String> fut = new CompletableFuture<>();
        new Thread(() -> {
            fut.thenAccept(v1 -> {
                try { Thread.sleep(1000L); } catch (InterruptedException e) {}
                System.out.println("thread-name-v1 " + Thread.currentThread().getName()); });

            fut.thenAccept(v1 -> {
                try { Thread.sleep(1000L); } catch (InterruptedException e) {}
                System.out.println("thread-name-v1 " + Thread.currentThread().getName()); });

            fut.thenAccept(v1 -> {
                try { Thread.sleep(1000L); } catch (InterruptedException e) {}
                System.out.println("thread-name-v1 " + Thread.currentThread().getName()); });

            fut.thenAccept(v1 -> {
                try { Thread.sleep(1000L); } catch (InterruptedException e) {}
                System.out.println("thread-name-v1 " + Thread.currentThread().getName()); });

            fut.thenAccept(v1 -> {
                try { Thread.sleep(1000L); } catch (InterruptedException e) {}
                System.out.println("thread-name-v1 " + Thread.currentThread().getName());});

        }, "thread-1").start();

        Thread.sleep(2000L);
        new Thread(() -> fut.complete("111"), "thread-2").start();

        new Thread(() -> fut.thenAccept(str -> {}), "thread-3").start();
    }
```
上面的例子println是否啥？你猜到了吗？
答案是：thread-2 或者 thread-3。
而正常我们会认为要么是thread-1（异步结果已完成，立即执行操作符），要么thread-2（设置结果时，处理操作符）

2. zipWith操作符 == thenCombine
当前的CompletableFuture和other CompletableFuture同时完成时，执行操作符
```java
    public <U,V> CompletableFuture<V> thenCombine
        (CompletionStage<? extends U> other,
         BiFunction<? super T,? super U,? extends V> fn) {
        return doThenCombine(other.toCompletableFuture(), fn, null);
    }
    
    //需要当前的CompletableFuture和other同时完成时，才调用操作符
    private <U,V> CompletableFuture<V> doThenCombine
        (CompletableFuture<? extends U> other,
         BiFunction<? super T,? super U,? extends V> fn,
         Executor e) {
        if (other == null || fn == null) throw new NullPointerException();
        CompletableFuture<V> dst = new CompletableFuture<V>();  //还是跟上面分析的一样，最后它
        ThenCombine<T,U,V> d = null;
        Object r, s = null;
        
        //当任意其中一个异步结果未完成时，进入if块
        if ((r = result) == null || (s = other.result) == null) {
            d = new ThenCombine<T,U,V>(this, other, fn, dst, e);
            CompletionNode q = null, p = new CompletionNode(d);
            while ((r == null && (r = result) == null) ||
                   (s == null && (s = other.result) == null)) {
                
                //第一次进来时，q == null，先执行 else-if
                if (q != null) {
                    if (s != null ||
                        UNSAFE.compareAndSwapObject
                        (other, COMPLETIONS, q.next = other.completions, q))  //other异步结果未完成时，挂载到操作符链表上
                        break;
                }
                else if (r != null ||
                         UNSAFE.compareAndSwapObject
                         (this, COMPLETIONS, p.next = completions, p)) {  //当前异步结果未完成时，挂载到操作符链表上
                    if (s != null)
                        break;
                    q = new CompletionNode(d);
                }
            }
        }
        
        //当两个异步结果都完成时，且操作符还未执行时，进入if块
        if (r != null && s != null && (d == null || d.compareAndSet(0, 1))) {
            T t; U u; Throwable ex;
            if (r instanceof AltResult) { //第一个异步结果 异常完成或null
                ex = ((AltResult)r).ex;
                t = null;
            }
            else {
                ex = null;
                @SuppressWarnings("unchecked") T tr = (T) r;
                t = tr;
            }
            if (ex != null) //说明第一个结果异常了，那么就不用去获取第二个异步结果的结果了。
                u = null;
            else if (s instanceof AltResult) {
                ex = ((AltResult)s).ex;
                u = null;
            }
            else {
                @SuppressWarnings("unchecked") U us = (U) s;
                u = us;
            }
            V v = null;
            if (ex == null) { //当两个异步结果都正常完成时，执行操作符
                try {
                    if (e != null) //上面分析过了
                        execAsync(e, new AsyncCombine<T,U,V>(t, u, fn, dst));
                    else
                        v = fn.apply(t, u);
                } catch (Throwable rex) {
                    ex = rex;
                }
            }
            //最终把结果设置到新的异步结果上
            if (e == null || ex != null) 
                dst.internalComplete(v, ex);
        }
        helpPostComplete();
        other.helpPostComplete();
        return dst;
    }

```
thenCombine方法，会到操作符的操作ThenCombine同时挂载到两个异步结果的操作符链表中。通过CAS控制，只有一次机会执行操作符操作。
ThenCombine跟if块的代码类似，就不展开说了。

3. (ambWith && map) == applyToEither
当前的CompletableFuture跟other CompletableFuture任意一个完成时，执行操作符。
```java
    public <U> CompletableFuture<U> applyToEither
        (CompletionStage<? extends T> other,
         Function<? super T, U> fn) {
        return doApplyToEither(other.toCompletableFuture(), fn, null);
    }
    
    private <U> CompletableFuture<U> doApplyToEither
            (CompletableFuture<? extends T> other,
             Function<? super T, U> fn,
             Executor e) {
            if (other == null || fn == null) throw new NullPointerException();
            CompletableFuture<U> dst = new CompletableFuture<U>();
            ApplyToEither<T,U> d = null;
            Object r;
            
            //当前异步结果和other 异步结果都未完成时，进入if块
            if ((r = result) == null && (r = other.result) == null) {
                d = new ApplyToEither<T,U>(this, other, fn, dst, e);
                CompletionNode q = null, p = new CompletionNode(d);
                while ((r = result) == null && (r = other.result) == null) {
                    if (q != null) {
                        if (UNSAFE.compareAndSwapObject
                            (other, COMPLETIONS, q.next = other.completions, q))
                            break;
                    }
                    //先挂载到当前异步结果上，
                    else if (UNSAFE.compareAndSwapObject
                             (this, COMPLETIONS, p.next = completions, p))
                        q = new CompletionNode(d);
                }
            }
            //此时，r可能来自result，也有可能来自other.result。
            //当两个异步结果都未完成时且操作符未执行过。
            if (r != null && (d == null || d.compareAndSet(0, 1))) {
                //到这里就跟doThenApply基本一样了，就不展开分析了
                T t; Throwable ex;
                if (r instanceof AltResult) {
                    ex = ((AltResult)r).ex;
                    t = null;
                }
                else {
                    ex = null;
                    @SuppressWarnings("unchecked") T tr = (T) r;
                    t = tr;
                }
                U u = null;
                if (ex == null) {
                    try {
                        if (e != null)
                            execAsync(e, new AsyncApply<T,U>(t, fn, dst));
                        else
                            u = fn.apply(t);
                    } catch (Throwable rex) {
                        ex = rex;
                    }
                }
                if (e == null || ex != null)
                    dst.internalComplete(u, ex);
            }
            helpPostComplete();
            other.helpPostComplete();
            return dst;
        }
```
3. flatMap == thenCompose
```java
    public <U> CompletableFuture<U> thenCompose
        (Function<? super T, ? extends CompletionStage<U>> fn) {
        return doThenCompose(fn, null);
    }
    
    private <U> CompletableFuture<U> doThenCompose
            (Function<? super T, ? extends CompletionStage<U>> fn,
             Executor e) {
            if (fn == null) throw new NullPointerException();
            CompletableFuture<U> dst = null;
            ThenCompose<T,U> d = null;
            Object r;
            
            //当前结果未完成时，尝试挂载到当前异步结果上
            if ((r = result) == null) {
                dst = new CompletableFuture<U>();
                CompletionNode p = new CompletionNode
                    (d = new ThenCompose<T,U>(this, fn, dst, e));
                while ((r = result) == null) {
                    if (UNSAFE.compareAndSwapObject
                        (this, COMPLETIONS, p.next = completions, p))
                        break;
                }
            }
            
            //当前异步结果已完成时，且操作符未执行过
            if (r != null && (d == null || d.compareAndSet(0, 1))) {
                T t; Throwable ex;
                if (r instanceof AltResult) {
                    ex = ((AltResult)r).ex;
                    t = null;
                }
                else {
                    ex = null;
                    @SuppressWarnings("unchecked") T tr = (T) r;
                    t = tr;
                }
                //异步结果，正常完成
                if (ex == null) {
                    if (e != null) {
                        if (dst == null)
                            dst = new CompletableFuture<U>();
                        execAsync(e, new AsyncCompose<T,U>(t, fn, dst));
                    }
                    else {
                        try {
                            //返回新的异步结果，正常情况下，doThenCompose方法返回的是这个异步结果。
                            CompletionStage<U> cs = fn.apply(t); 
                            if (cs == null ||
                                (dst = cs.toCompletableFuture()) == null)
                                ex = new NullPointerException();
                        } catch (Throwable rex) {
                            ex = rex;
                        }
                    }
                }
                
                if (dst == null)
                    dst = new CompletableFuture<U>();
                //异步结果异常完成时，或者执行操作符时发生异常，通知新的异步结果。
                if (ex != null)
                    dst.internalComplete(null, ex);
            }
            helpPostComplete();
            dst.helpPostComplete();
            return dst;
        }
```
4. 后面的exceptionally，whenComplete，handle，实现就跟thenApply差不多。这里就不展开说了。
   CompletableFuture源码的核心流程都已经覆盖完了。
5. 即使CompletionStage，CompletableFuture怎么怎么不好，但没办法，这是标准库的类（亲儿子）。所以做好跟标准库的兼容还是有必要的。可以很方便和其他实现CompletionStage的框架或类库做集成。

写了点异步相关的基础库，之前想直接继承CompletableFuture做拓展就行了。后面发现了上面所说的它回调机制的坑，
所以就自己重写了一个CompletionStage。
总体上，还是非常简单的。基于回调(addHandler)的方式，把上面的所有操作符都实现了。源码在这里。[CompletableResult](https://github.com/wang007/pandora/blob/master/pandora-ext/src/main/java/com/github/pandora/asyncResult/CompletableResult.java)

--- 
还是上一篇文章说过，异步往往伴随着回调。而且还举了一个回调嵌套（callback hell）的例子，这个我们就会CompletableFuture改善一下。
```java
startServer(socket -> {
            socket.read()
                    .thenCompose(str -> {
                        System.out.println("result -> " + str);
                        return socket.write("1");
                    })
                    .thenCompose(v -> {
                        return socket.write("2");
                    })
                    .thenCompose(v -> {
                        return socket.write("3");
                    })
                    .whenComplete((v, err) -> {
                        System.out.println("socket close");
                        socket.close();
                    });
        });
```
这个看起来是不是简洁了很多。callback hell会被平铺（flat）了。这个代码跟上篇文章的回调嵌套执行结果是一样的。


下面是如何实现的。可以看到改动也是非常小的，但是对使用方的效果完成不一样，可阅读性好非常多。
```java
	public CompletionStage<String> read() {
            CompletableFuture<String> future = new CompletableFuture<>();
            read(obj -> {
                if(obj instanceof String) {
                    future.complete((String) obj);
                } else {
                    future.completeExceptionally(obj == null ? new NullPointerException() : (Throwable) obj);
                }
            });
            return future;
        }

        public CompletionStage<Void> write(String data) {
            CompletableFuture<Void> future = new CompletableFuture<>();
            write(data, obj -> {
                if(obj == null) {
                    future.complete(null);
                } else {
                    future.completeExceptionally(obj == null ? new NullPointerException() : (Throwable) obj);
                }
            });
            return future;
        }

```
这个例子，包含前面说到的自实现的CompletionStage，ListenableFuture也好，都是通过回调扩展出来的，所以只有回调这最原始的api，就可以做无限的拓展。


#### 所以异步非阻塞框架，都是基于这种回调的方式，然后通过各种手段不断地去解决回调嵌套的问题。下面即将讲到的响应式编程，协程，也是如此。


总结：
1. Future的源码分析。
2. 如何基于Future，改成基于回调机制的Future。
3. CompletionStage，CompletableFuture的源码分析，分类总结。
4. 如何简单的实现一个CompletionStage。([pandora](https://www.github.com/wang007/pandora))
5. CompletionStage使用方式和如何改善callback hell。

--- 
好了，晚安。

































