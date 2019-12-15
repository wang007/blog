---
title: java异步编程探究-响应式编程（ReactiveX）
date: 2019-12-07 02:29:44
tags:
 - java
 - 异步
 - reactive
 - 响应式编程
categories:
 - java
 - 异步
 - 响应式编程
 - reactive
--- 

该文章是Java异步编程探究系列文章的第三章。

### 前言

> 响应式编程，也有不少人叫反应式。不管是响应式编程还是反应式编程，反正说的就是它。这里我不纠结于它叫啥。管它白猫还是黑猫，能打老鼠的就是好猫。所以这里都统一叫响应式编程。

相信你如果是第一次接触它的话，很懵逼。到底是干啥的啊，有什么用啊。我当初也是一样。就像当初学异步一样多疑问，什么是响应式编程，为什么这样就是响应式编程啊，等等。


先来看看reactivex官网是如何定义的。
> An API for asynchronous programming with observable streams 一个异步编程与可观察流的API（有道翻译）

再来看看project Reactor官网是如何定义的。
> Reactor is a fourth-generation Reactive library for building non-blocking applications on the JVM based on the Reactive Streams Specification  Reactor是第四代反应库，用于基于反应流规范在JVM上构建非阻塞应用程序(有道翻译)

这两个定义是非常简练的，以至于我当初用Rxjava一段时间之后，也没明白什么是响应式编程，直到我看到这个解释：
#### 响应式编程是一种面向异步数据流变化和传播的编程范式。
按照这个定义，把关键字提取出来
1. 面向异步数据流
2. 变化和传播
3. 编程范式

#### 1.面向异步数据流
> 首先的是异步，得在异步环境才有使用响应式的必要。同步不是说不行，同步直接返回raw对象就行了，没必要再用Reactive对象去包装了。

#### 2.变化和传播
> 响应式提供了丰富的操作符对响应式对象进行变换。例如 map，flatMap，filter等等。这个就跟Java8提供的Stream中的操作符一样。

#### 3. 编程范式
> 这个可以类比数据库范式，数据库范式就很熟悉了吧，它可以帮助更有效的组织数据且设计表的一个参考。但是我们往往有时候为了提高查询效率，往往会做一些适当的数据冗余。所以说范式只是一个参考，一种规范吧，是充分非必要的。编程范式也是类似的，遵守这个范式可以让代码的可阅读性大大增强。callback hell的可阅读性就非常的差。

实际上按照响应式的定义，CompletableFuture也是响应式对象。也就是说使用CompletableFuture编程的话也可以叫做响应式编程。只不过CompletableFuture的数据流是单一的，只能发射一个对象。
跟Rxjava#Single，ProjectReactor#Mono类似。

#### 背压（Back Pressure）
背压：有些地方也翻译成反压，不过这个不重要。
首先得知道为什么会诞生反压？
假如上下游在不同的线程，上游产生10条数据，下游处理10条。上游产生100条，下游处理100条，没有问题。
但是当下游线程处理数据能力不及上游线程生产数据能力时，就会导致下游处理不过来，数据堆积等最后可能导致OOM。而背压正是解决这种问题提出的一种概念，用于协调上下游。
背压通过某一机制让下游通知到上游产生多少数据，按需所求。处理完之后可以再通知上游产生数据。

#### 响应式宣言
响应式宣言是一种软件架构理念，遵守这种理念实现的应用软件，那么就称之为响应式系统。
那响应式编程算不算响应式宣言提到的响应式系统时，我觉得算。
响应式宣言提到的4个特质（即时响应性，回弹性（容错性），弹性，消息驱动）在响应式编程中都有实现。
即时响应性：可通过timeOut的方式几十响应。
回弹性：在subscribe时，对错误结果的处理。
弹性：背压。
消息驱动：把异步数据流也是一种消息。
很显然用响应式编程构建的系统也可以称为响应式系统。


### 正文
在Java中，响应式编程用的比较多的是Rxjava，而作为后起之秀的Project Reactor也非常不错，而且Project Reactor还作为构建spring webflux应用的核心工具。
实际上Rxjava和Project Reactor的代码相似度很大，所以后面我可能就挑选其中一个来深入讲解。
Rxjava和Project Reactor都实现了响应式流的规范[Reactive Streams](https://www.reactive-streams.org)，但是只有Rxjava2的Flowable实现了规范，而Single，Observable，Maybe，Completable都没有实现规范，也就是说只有Flowable支持背压。
相比之下，Project Reactor只有两个核心对象Mono，Flux，而且都实现了响应式规范。从这一点来讲，我觉得Project Reactor做的更好一点，所以接下来会以Project Rector作为核心讲解。


直接看Reactive Streams几个核心对象。
数据源
```java
        public interface Publisher<T> {
            public void subscribe(Subscriber<? super T> s);
        }
```
这个数据源对象非常的干练，只有一个subscribe方法。
subscribe就是订阅这个数据源。
Publisher可以多次订阅。
#### 不管Project Reactor的Mono，Flux，还有Rxjava的Flowable，Single，Observable，Maybe，Completable，这些数据源都是冷数据源，也就是只有真正subscribe它们的时候跟才会产生数据。这一点跟Java8 Stream很相似。

订阅者 Subscriber
```java
        public interface Subscriber<T> {
            
            //当subscribe数据源时调用，只会执行一次。
            public void onSubscribe(Subscription s);

            //数据源发射的数据
            public void onNext(T t);

            //当出现错误时通知（终止状态，跟onComplete方法互斥，即只会调用两者中的其中一个）
            public void onError(Throwable t);

            //当数据源发射完全部数据时调用（终止状态，跟onError方法互斥，即只会调用两者中的其中一个）
            public void onComplete();
        }
```

代表数据源和订阅者之间的关联对象 Subscription
订阅一次，产生一个独立的关联对象 Subscription
这个对象是实现背压的关键。
```java
    public interface Subscription {

        public void request(long n);

        public void cancel();
    }
```
其中request方法向数据源请求数据，参数n是数据的数量。cancel方法就是取消订阅，那么数据源（上游）将不再发射数据，即使调用request方法请求数据。
所以通过request方法是实现背压的关键，通过调用requeset方法向数据源（上游）请求发射数据，当发射的数据处理完后再调用request继续发射。

实际上这3个对象就能构成响应式流的核心对象了，但是在介绍Publisher时提到这些数据流默认都是冷的，如果想要热的怎么办？热的就是说立即发射数据，而不用等到subscribe时再发射。
Processor
```java
    public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

好了，响应式流的规范就说完了。


###Project Reactor

#### Mono
可能发射1个数据（调用onNext一次），也可能不发射数据（不调用onNext）的Publisher。实际上跟Rxjava2的Maybe是差不多的，只不过Maybe没有背压。

#### Flux
发射0个或多个数据的的Publisher。跟Rxjava2的Flowable很相似。


栗子

mono
```java
            Mono.just("name")                       //创建发射单个数据的数据源Mono
                .map(str -> str + ": wang007")      //对数据进行转换
                .flatMap(str -> {                   //平铺
                    System.out.println("str -> " + str);
                    return Mono.just(str.length());
                })
                .map(len -> len + "" + len)                //再转换，转换成String的长度
                .filter(lenStr -> lenStr.length() > 5)     //过滤
                .subscribe(str -> {                        //订阅
                    System.out.println("result -> " + str);
                }, err -> {
                    System.out.println("onError");
                    err.printStackTrace();
                }, () -> {
                    System.out.println("onComplete");
                });
```
例子很简单也只是使用了一些非常简单的操作符。


flux
```java
            Flux.just("s1", "s2", "s3")
                .map(str -> str + str)
                .flatMap(str -> {
                    System.out.println("str -> " + str);
                    if(str.endsWith("1")) return Flux.just("s1-1");
                    else if(str.endsWith("2")) return Flux.just("s2-1", "s2-2");
                    else return Flux.just("s3-1", "s3-2", "s3-3");
                })
                .filter(str -> str.endsWith("1"))
                .subscribe(str -> {
                    System.out.println("result -> " + str);
                }, err -> {
                    System.out.println("onError");
                    err.printStackTrace();
                }, () -> {
                    System.out.println("onComplete");
                });

```
还是很简单，跟上面的例子差不多。


--- 

下面就有Reactor来解决问题。

本系列第一篇文章有段回调嵌套的代码，现在用响应式编程的方式解决掉回调的问题。
```java
        startServer(socket -> {
            socket.rxRead()
                    .flatMap(str -> {
                        System.out.println("result -> " + str);
                        return socket.rxWrite("1");
                    })
                    .flatMap(v -> {
                        return socket.rxWrite("2");
                    })
                    .flatMap(v -> {
                        return socket.rxWrite("3");
                    })
                    .subscribe(v -> {
                        System.out.println("socket close");
                        socket.close();
                    }, err -> {
                        System.out.println("socket close");
                        socket.close();
                    }, () -> {
                        System.out.println("socket close");
                        socket.close();
                    });
        });

```
将callback hell flat之后，就变成这样子，看起来清晰很多，每一个continuatin都是flatMap拼接在后面。这个嵌套就只有一层。
实际上这个看起来跟上一篇文章CompletableFuture非常相似。


看看怎么实现的。
```java
       public Mono<String> rxRead() { //调用的时候先返回一个Mono对象
            return new CallbackMono<>(1, consumer -> {
                Consumer<? extends Object> co = consumer;
                read((Consumer<Object>) co);
            });
        }

        public Mono<Void> rxWrite(String data) {
            CallbackMono<Object> mono = new CallbackMono<>(2, consumer -> {
                write(data, consumer);
            });
            return mono.map(v -> null);
        }
```

```java
public class CallbackMono<T> extends Mono<T> {

    private Consumer<Consumer<T>> apply;
    private int readOrWrite;

    public CallbackMono(int readOrWrite, Consumer<Consumer<T>> apply) {
        this.apply = apply;
        if(readOrWrite != 1 && readOrWrite != 2) throw new IllegalArgumentException("readOrWrite");
        this.readOrWrite = readOrWrite;
    }

    @Override
    public void subscribe(CoreSubscriber<? super T> actual) {  
        try {
            apply.accept(t -> {
                if(readOrWrite == 1) { //read
                    if(t instanceof String) {
                        actual.onSubscribe(Operators.scalarSubscription(actual, t));
                    } else {
                        Operators.error(actual, (Throwable) t);
                    }
                } else { //write
                    if(t == null) {
                        Operators.complete(actual);
                    } else {
                        Operators.error(actual, (Throwable) t);
                    }
                }
            });
        } catch (Throwable e) {
            Operators.error(actual, e);
        }
    }
}
```
在调用的时候，先返回一个Mono对象，并且把把这个方法所进行的行为先保存起来，例如rxRead方法，先把read的行为保存下来，在CallbackMono#subscribe时，才执行对应的read行为，read方法的回调结果用于通知到subscriber上。


--- 
用于篇幅有限，project Reactor实现原理将放到下篇文章。



参考文章
[响应式宣言](https://www.reactivemanifesto.org)
[Reactive Stream](https://www.reactive-streams.org)
[ReactiveX](http://reactivex.io)
[Project Reactor](https://projectreactor.io)
[什么是响应式编程](https://www.jianshu.com/p/111e0a4b9b17?utm_campaign=hugo)








































