---
title: java异步编程探究-Project Reactor实现原理-2
date: 2019-12-29 00:06:12
tags:
 - java
 - 异步
 - reactive
 - Project Reactor
 - 响应式编程
categories:
 - java
 - 异步
 - 响应式编程
 - reactive
---

## 零
接着上一篇文章继续分析。

实际上，理解上融合模式的思路之后，源码看看起来就清晰很多。特别是异步融合模式，通过onNext方法向下游通知数据已准备好，很容易理解为往下游发射数据。

<!-- more -->

---
### publishOn操作符
![Mono publishOn](./java-async-road-4/MonoObserveOn.png)
> 这里用了Rxjava2中的ObserveOn操作符，但是效果是一样的。
图片描述已经很清楚了，就是下游执行在scheduler线程上。

```java
//很明显这里没有对上游做融合处理
//入参指定scheduler
public final Mono<T> publishOn(Scheduler scheduler) {
	//这个if分支属于优化分支，不深入分析
	if(this instanceof Callable) {
		if (this instanceof Fuseable.ScalarCallable) {
			try {
				T value = block();
				return onAssembly(new MonoSubscribeOnValue<>(value, scheduler));
			}
			catch (Throwable t) {
				//leave MonoSubscribeOnCallable defer error
			}
		}
		Callable<T> c = (Callable<T>)this;
		return onAssembly(new MonoSubscribeOnCallable<>(c, scheduler));
	}
	
	return onAssembly(new MonoPublishOn<>(this, scheduler));
}
```
最后创建一个MonoPublishOn实例
```java
final class MonoPublishOn<T> extends InternalMonoOperator<T, T> {

	final Scheduler scheduler;

	MonoPublishOn(Mono<? extends T> source, Scheduler scheduler) {
		super(source);
		this.scheduler = scheduler;
	}

	//
	@Override
	public CoreSubscriber<? super T> subscribeOrReturn(CoreSubscriber<? super T> actual) {
		return new PublishOnSubscriber<T>(actual, scheduler);
	}

	@Override
	public Object scanUnsafe(Attr key) {
		if (key == Attr.RUN_ON) return scheduler;

		return super.scanUnsafe(key);
	}
}
```
MonoPublishOn没有实现Fuseable接口，所以不支持融合的
InternalMonoOperator上一篇文章分析过了，这里包装了下游的Subscriber（actual），接着分析PublishOnSubscriber
```java
static final class PublishOnSubscriber<T>
			implements InnerOperator<T, T>, Runnable {

		final CoreSubscriber<? super T> actual;

		final Scheduler scheduler;

		Subscription s;

		volatile Disposable future;

		volatile T value;
		

		volatile Throwable error;

		PublishOnSubscriber(CoreSubscriber<? super T> actual,
				Scheduler scheduler) {
			this.actual = actual;
			this.scheduler = scheduler;
		}

		//第1步 往下游传递Subscription，下游会调用到request方法
		@Override
		public void onSubscribe(Subscription s) {
			if (Operators.validate(this.s, s)) {
				this.s = s;

				actual.onSubscribe(this);
			}
		}

		//第2步 直接把请求消息往上游传递，上游会调用onNext方法
		@Override
		public void request(long n) {
			s.request(n);
		}

		//第3步 保存值，并且在scheduler调度执行发射数据到下游
		@Override
		public void onNext(T t) {
			value = t;
			trySchedule(this, null, t);
		}

		@Override
		public void onError(Throwable t) {
			error = t;
			trySchedule(null, t, null);
		}

		@Override
		public void onComplete() {
			if (value == null) { //没有调度过
				trySchedule(null, null, null);
			}
		}

		//第4步 
		void trySchedule(
				@Nullable Subscription subscription,
				@Nullable Throwable suppressed,
				@Nullable Object dataSignal) {

				if(future != null){
					return;
				}

				try {
					//this实现了Runnable，最后会在scheduler的线程池上执行run方法
					future = this.scheduler.schedule(this);
				}
				catch (RejectedExecutionException ree) {
					actual.onError(Operators.onRejectedExecution(ree, subscription,
							suppressed,	dataSignal, actual.currentContext()));
				}
		}

		//第5步 
		//当执行此run方法时，执行线程在scheduler上的线程。
		@Override
		public void run() {
			if (OperatorDisposables.isDisposed(future)) {
				return;
			}
			//value在onNext方法设置了，设置为null，代表已处理
			T v = (T)VALUE.getAndSet(this, null);

			//调用下游的方法，此时已执行在scheduler上的线程，完全符合observeOn图的定义。
			if (v != null) {
				actual.onNext(v);
				actual.onComplete();
			}
			else {
				Throwable e = error;
				if (e != null) {
					actual.onError(e);
				}
				else {
					actual.onComplete();
				}
			}
		}

		//省略若干代码
	}
```
---
### subscribeOn操作符
![Mono SubscribeOn](./java-async-road-4/MonoSubscribeOn.png)
根据图片描述，订阅上游执行scheduler上
```java
public final Mono<T> subscribeOn(Scheduler scheduler) {
		//优化分支，忽略
		if(this instanceof Callable) {
			if (this instanceof Fuseable.ScalarCallable) {
				try {
					T value = block();
					return onAssembly(new MonoSubscribeOnValue<>(value, scheduler));
				}
				catch (Throwable t) {
					//leave MonoSubscribeOnCallable defer error
				}
			}
			@SuppressWarnings("unchecked")
			Callable<T> c = (Callable<T>)this;
			return onAssembly(new MonoSubscribeOnCallable<>(c,
					scheduler));
		}
		return onAssembly(new MonoSubscribeOn<>(this, scheduler));
}
```
这个也是不支持融合的，直接返回MonoSubscribeOn。
```java
final class MonoSubscribeOn<T> extends InternalMonoOperator<T, T> {

	final Scheduler scheduler;

	MonoSubscribeOn(Mono<? extends T> source, Scheduler scheduler) {
		super(source);
		this.scheduler = scheduler;
	}

	@Override
	public CoreSubscriber<? super T> subscribeOrReturn(CoreSubscriber<? super T> actual) {
		Scheduler.Worker worker = scheduler.createWorker();

		//同样也是Runnable
		SubscribeOnSubscriber<T> parent = new SubscribeOnSubscriber<>(source,
				actual, worker);
		actual.onSubscribe(parent);	//这里调用下游的onSubscribe方法

		try {
			//在scheduler执行
			worker.schedule(parent);
		}
		catch (RejectedExecutionException ree) {
			if (parent.s != Operators.cancelledSubscription()) {
				actual.onError(Operators.onRejectedExecution(ree, parent, null, null,
						actual.currentContext()));
			}
		}
		//前面InternalMonoOperator的时候提到了，return null代表subscribeOrReturn方法已经处理了对上游的subscribe
		//所以InternalMonoOperator#subscribe会停止往上游包装Subscriber，
		//所以这里的入参actual被多个下游包装过的Subscriber。不清楚的话继续看InternalMonoOperator#subscribe方法
		return null;
	}

	@Override
	public Object scanUnsafe(Attr key) {
		//返回执行所在的scheduler
		if (key == Attr.RUN_ON) return scheduler;
		return super.scanUnsafe(key);
	}
}
```
那么SubscribeOnSubscriber是实现的关键
```java
	static final class SubscribeOnSubscriber<T>
			implements InnerOperator<T, T>, Runnable {

		final CoreSubscriber<? super T> actual;	//下游Subscriber

		final Publisher<? extends T> parent; //上游

		final Scheduler.Worker worker; //调度器worker

		volatile Subscription s;	

		volatile long requested; 	//下游请求的信号量
		
		volatile Thread thread;		//

		//省略AtomicReferenceFieldUpdater						
		//省略构造方法
		
		//根据subscribeOrReturn方法的处理。有两种情况
		//1. run方法先执行，request方法后执行
		//2. request方法先执行，run方法后执行
		//不过这里顺序不是重点

		//第1步， 订阅上游执行在worker线程上
		@Override
		public void run() {
			//保存当前线程引用
			THREAD.lazySet(this, Thread.currentThread());
			//订阅上游，很明显订阅上游执行在Scheduler.Worker上
			parent.subscribe(this);
		}

		//第2步，因为对上游subscribe时，subscribe方法时会调用onSubscribe，所以onSubscribe方法也会执行在worker线程上，
		//但是不是绝对执行在worker线程上，因为上游在subscribe方法时也有可能做切换线程的操作
		@Override
		public void onSubscribe(Subscription s) {
			if (Operators.setOnce(S, this, s)) {
				long r = REQUESTED.getAndSet(this, 0L);
				if (r != 0L) {	//说明下游调用过request方法了
					trySchedule(r, s);
				}
			}
		}

		//也有可能request才是第1步
		@Override
		public void request(long n) {
			if (Operators.validate(n)) {
				Subscription a = s;
				if (a != null) { //有上游的Subscription，那么肯定是先执行了run方法了，
					trySchedule(n, a);
				}
				else { //先执行了request，那么request请求数据量先保存下来
					Operators.addCap(REQUESTED, this, n);	
					a = s;
					if (a != null) { //再检查一下
						long r = REQUESTED.getAndSet(this, 0L);
						if (r != 0L) {
							trySchedule(n, a);
						}
					}
				}
			}
		}

		//以上两种情况最后都会执行到trySchedule方法中来，该方法是向上游调用request方法
		第3步
		void trySchedule(long n, Subscription s) {
			//在onSubscribe方法执行的trySchedule方法情况
			//因为都在worker线程上，所以直接调用上游的request方法即可
			if (Thread.currentThread() == THREAD.get(this)) {
				s.request(n);
			}
			else {
				//在request方法执行的trySchedule方法情况，此时不在worker线程上，切回到worker线程上
				try {
					worker.schedule(() -> s.request(n));

				}
				catch (RejectedExecutionException ree) {
					if (!worker.isDisposed()) {
						actual.onError(Operators.onRejectedExecution(ree,
								this,
								null,
								null,
								actual.currentContext()));
					}
				}
			}
		}

		//第4步，调用上游的request方法执行在worker线程上，request方法可能包括**直接**向下游发射onNext方法。
		//所以onNext方法也有可能执行worker线程上
		@Override
		public void onNext(T t) {
			actual.onNext(t);
		}
		@Override
		public void onError(Throwable t) {
			try{
				actual.onError(t);
			}
			finally {
				worker.dispose();
				THREAD.lazySet(this,null);
			}
		}


		//省略若干方法
	}

```
实际上面分析trySchedule方法内调用上游的request方法，但是request方法是否包括直接向下游发射数据的onNext方法是不一定的。
例如：
- publishOn操作符，request方法直接往上游传，上游调用publishOn的onNext方法，onNext方法内却切换了publishOn所在的Scheduler线程。
- flatMap操作符，request方法到flatMap的subscription时，也得在新数据源的数据是否完成时才执行下游的onNext方法

总结：
- subscribeOn操作符：明确的是对调用上游subscribe和request所在的Scheduler，正如它的操作符名字一样subscribeOn，subscribe在哪个Scheduler。
- publishOn操作符：明确的是对调用下游发射数据（onNext）和发射通知（onComplete、onError）所在的Scheduler。

---
### subscribe
最后再来说说Mono#subscribe方法
subscribe方法，入参是lambda类型的，最后都会生成一个coreSubscriber.
例如：
```java
public final Disposable subscribe(
			@Nullable Consumer<? super T> consumer,
			@Nullable Consumer<? super Throwable> errorConsumer,
			@Nullable Runnable completeConsumer) {
		return subscribe(consumer, errorConsumer, completeConsumer, (Context) null);
}
```
最后都会执行到
```java
public final Disposable subscribe(
			@Nullable Consumer<? super T> consumer,
			@Nullable Consumer<? super Throwable> errorConsumer,
			@Nullable Runnable completeConsumer,
			@Nullable Context initialContext) {
		return subscribeWith(new LambdaMonoSubscriber<>(consumer, errorConsumer,
				completeConsumer, null, initialContext));
}

//subscribeWith很简单，就是执行subscribe方法，最后返回Disposable 
//LambdaMonoSubscriber实现Disposable方法
public final <E extends Subscriber<? super T>> E subscribeWith(E subscriber) {
		subscribe(subscriber);
		return subscriber;
}
```
调用subscribe方法时，如果上游是OptimizableOperator的话，会执行到subscribeOrReturn方法，然后会对最开始的LambdaMonoSubscriber实例包装。
还有一个subscribe方法，入参是subscriber的，是Reactive Stream规范中的Publisher定义的方法。
```java
public final void subscribe(Subscriber<? super T> actual) {
		CorePublisher publisher = Operators.onLastAssembly(this);
		//最后还是包装成CoreSubscriber
		CoreSubscriber subscriber = Operators.toCoreSubscriber(actual);

		//这里的代码就跟InternalMonoOperator#subscribe方法基本一样了
		if (publisher instanceof OptimizableOperator) {
			OptimizableOperator operator = (OptimizableOperator) publisher;
			while (true) {
				subscriber = operator.subscribeOrReturn(subscriber);
				if (subscriber == null) {
					// null means "I will subscribe myself", returning...
					return;
				}

				OptimizableOperator newSource = operator.nextOptimizableSource();
				if (newSource == null) {
					publisher = operator.source();
					break;
				}
				operator = newSource;
			}
		}

		//一样包装到数据源头了
		publisher.subscribe(subscriber);
}
```
LambdaMonoSubscriber比较简单，就不分析了，里面有一点需要关注的是onSubscribe方法。一般情况下，不需要覆写onSubscribe方法，默认是拉满的。
```java
//也实现了Disposable
final class LambdaMonoSubscriber<T> implements InnerConsumer<T>, Disposable {
	public final void onSubscribe(Subscription s) {
		if (Operators.validate(subscription, s)) {
			this.subscription = s;
			
			//所以覆写的，就不要忘记调用Subscription#request方法
			if (subscriptionConsumer != null) {	
				try {
					subscriptionConsumer.accept(s);
				}
				catch (Throwable t) {
					Exceptions.throwIfFatal(t);
					s.cancel();
					onError(t);
				}
			}
			else {
				//默认情况
				s.request(Long.MAX_VALUE);
			}

		}
	}

	//dispose方法实际上也是对上游调用cancel方法
	public void dispose() {
		Subscription s = S.getAndSet(this, Operators.cancelledSubscription());
		if (s != null && s != Operators.cancelledSubscription()) {
			s.cancel();	
		}
	}

	//省略若干代码
}
```
---
### toProcessor
或许你应该没忘记，在Reactive Stream规范还有一个接口：Processor。
在介绍响应式编程第一篇文章中有介绍过它。它能让Publisher从冷数据变成热数据，下面就分析分析它是如何实现的。
```java
public final MonoProcessor<T> toProcessor() {
		MonoProcessor<T> result;
		if (this instanceof MonoProcessor) {
			result = (MonoProcessor<T>)this;
		}
		else {
			result = new MonoProcessor<>(this);
		}
		result.connect();
		return result;
}
```
在操作符内，会创建一个MonoProcessor实例。如果当前Mono本身是Processor，那直接使用。然后调用了connect方法。
```java
public final class MonoProcessor<O> extends Mono<O>
		implements Processor<O, O>, CoreSubscriber<O>, Disposable, Subscription, Scannable {

	void connect() {
		Publisher<? extends O> parent = source; //上游
		if (parent != null && SUBSCRIBERS.compareAndSet(this, EMPTY_WITH_SOURCE, EMPTY)) {
			parent.subscribe(this);
		}
	}
}
```
1. MonoProcessor继承了Mono，也实现了Processor
2. connect方法最后也是订阅了上游。所以Processor方法能让Publisher从冷数据变成热数据，上游的数据也会发射到MonoProcessor并暂存下来。
3. toProcessor操作符也仅仅是它所有上游变成热数据，对于其下游来说，还是冷数据。
例如
```java
    Mono.just("name")                       //创建发射单个数据的数据源Mono
         .map(str -> str + ": wang007")      //对数据进行转换
         .flatMap(str -> {                   //平铺
            System.out.println("str -> " + str);
             return Mono.defer(() -> Mono.just(str.length()));
          })
          .toProcessor()
          .map(len -> len + "" + len)                //再转换，转换成String的长度
          .filter(lenStr -> lenStr.length() > 5)     //过滤
          .subscribe();
```
toProcessor方法只会作用到上游的just、map、flatMap操作符，但是对下游的map、filter操作符则没有影响

本文分析基于3.3.0.RELEASE版本

Flux再下一篇文章说

参考文章：
- [reactiveX官网](http://reactivex.io/) 
- [project Reactor官网](http://projectreactor.io/) 

---



