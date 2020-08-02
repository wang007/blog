---
title: java异步编程探究普拉斯
date: 2019-11-26 21:40:07
tags: 
  - java
  - 异步
  - 协程

categories:
  - java
  - 异步
  - 协程
---

从技术圈的近些年来，异步相关的话题是一个绕不开的话题。前有node.js轻松hold住10万并发，后者golang脚踢Java。 
实际上呢，Java在高并发方面的性能吊打golang是没有问题的（口说无凭，上[性能比拼网站](http://www.techempower.com/benchmarks)）。奈何Java要写出高并发的应用复杂较大。而golang呢，有了协程的加持可以简单的写成高并发能力的应用。
而Java呢，其实也在往方面发力，Java的协程也正在磨刀霍霍向猪羊。[Project Loom](http://openjdk.java.net/projects/loom/) 这是一个非常值得期待的新特性。下面我会详细说说这个Project Loom.

因为之前分享过一次关于Java异步编程探究，这次直接+ PLUS

<!-- more -->

> 同时往往有些小伙伴看不清异步的真相，瞎JB异步。列举几个瞎JB异步的我例子。
> 1. 创建一个线程或线程池，然后把后续的代码扔到这个新线程里面，这种就是典型的瞎JB异步，对于性能提升基本没有，甚至反而更加慢了。
> 2. 在spring中使用@Async注解，它的实现方式更1.的差不多。
> 3. 在spring中使用DeferredResult，然后里面又是各种同步的写法。

#### 像这种，把后续的工作转移到其他线程的做法，我们统一叫做 **阻塞转移**。
> 因为这种本质上还是没有改变同步阻塞IO阻塞的本质，最终还是需要阻塞掉某个线程的。

--- 

> 正如我们所看到的那样，为啥“异步”会频繁出现在我们视线中？
java现在主流的同步编程模型已经越来越难以支撑现阶段互联网的大流量。过去Java依赖强大的多线程处理能力轻松满足当年的性能需求。 而现在也正在因为servlet的request/per thread 模型的效率低下，非常消耗机器，有效做功的比率低等问题。
以tomcat为例，虽然说tomcat8已经加入了nio模式，对于性能提升影响较小。tomcat默认NIO模式 = 1个acceptor + 2 IO worker + 200个业务线程。其实加入了NIO模式之后，tomcat原本的request/per thread模型依赖没有改变。所以即便改成这种模式，瓶颈往往不在acceptor和IO worker上。而是在业务线程池上。 

举个例子
> 例如一秒内，1000请求过来，但是接口处理得1秒（例如里面包含10个jdbc查询）。那么按照200个线程池比例，这一秒最多处理200个请求，其他请求入队列。这1000个请求对于accecpt/IO worker其实是很轻松就能搞定的。
而且还有一个重要的问题，200个线程肯定是不能同时跑的，因为cpu core数量就那么几个。所以这么多线程一起工作，一定会有大量的线程调度，阻塞，唤醒等线程切换的工作，而这些对于业务代码的工作来说没有意义。这些线程切换的越频繁，业务代码的有效做工就越少。同时线程是一个非常重量级的对象，它不可能创建成千上万的。而且线程数量也不是越多越好。恰好跑满cpu且少量的线程切换还是最理想的状态。

> 还是这个例子，为什么异步非阻塞框架就能轻松hold住这1000个请求呢？
>  - 因为这些不再使用request/req thread模型，同时也不再使用阻塞的BIO，全部替换成非阻塞的NIO。
>  - request（connection）只是一个普通的对象。
>  - 任何调用（IO调用）不再因IO阻塞而发生线程切换，线程一直在高效率的跑着。如此，线程数量一般情况下只需要cpu core * 2，这时性能最好。
>  - 因为所有调用（IO调用）不再是阻塞等待结果，所以需要一个callback函数来执行拿到结果之后的代码。
>  - 因为所有调用都是callback获取结果，所以你的代码会割裂到每个callback函数里，最后形成callback hell，这会使得编程复杂度大大提升。所以后面的CompletableFuture，reactive，coroutine，Fiber。这些技术基本上都是以解决这个callback为主。

--- 

#### 读到这里，有个先入为主的概念，异步会伴随着callback。


### 下面重点来说说异步这个东西


> 如果您已经懂了什么是异步，什么是响应式编程，什么是协程？以及它们各自的工作原理，那么这篇文章可能就不大适合您，尽请跳过吧。

 

说异步，又不得先扯扯同步。
#### 同步编程

- 同步编程的优点：代码顺序执行，逻辑清晰。方法结束也意味着计算完成并返回对应的结果。好处就是代码很容易理解，可读性高。这点无论对于企业开发，还是个人开发来说都非常重要。代码其一是为了让计算机运行，然后产生结果。其二代码需要迭代，需要别人能够快速理解并上手你的代码。
- 缺点也一样明显，正是因为其顺序执行，当遇到需要一些HTTP，JDBC查询时，也必须等待其结果返回才能继续往下执行。例如JDBC查询，里面使用了 **同步阻塞IO**，使得当前调用的线程阻塞等待，然后OS会调度该线程让出cpu执行时间片。让其他线程工作的线程继续工作，待IO返回之后线程阻塞唤醒，参与竞争时间片并往下之后执行代码。

因为过去以及现在，基本用的都是同步编程所以这个很容易理解。 下面我们重点来说说异步编程，以及其他大佬们在这条路上的探索成果。

> 一直以为我比较了解异步，但是每当别人问我什么是异步，为什么这么写就是异步时，我往往是一种有话说不出的感觉。所以如果以后别人问我这个问题时，我想贴上这篇文章让别人也能轻易的理解这个概念。


### 异步

- 我们经常所说的异步应用，异步非阻塞框架等，这个“异步”是指应用层面的异步，即异步方法调用不会阻塞线程且注册一个callback，在callback里回调通知你结果。从这个角度看，确实异步（通知机制）非阻塞（调用方是否阻塞）的。 
- 但是在IO层面，对应的OS层面来说应该同步非阻塞IO，同步非阻塞IO（IO多路复用）。所以Netty，Vert.x，Webflux等一众选手都应该是使用同步非阻塞IO的异步非阻塞框架/工具。 emmm... 这个有点绕。
所以一说异步非阻塞框架，一般指的是上层代码是异步非阻塞的，但是IO层面是同步非阻塞IO。

什么是异步IO？

####  不因IO而阻塞的IO就叫异步IO。 当然这是废话，如果我这么说，我估计你都会想打我。

当然饭还是得一口一口饭， 先来说说 阻塞/ 非阻塞，同步/ 异步。当然这个老生常谈的话题，网上一搜有说好的，有说烂的。其中往往烂的占据了大多数，而且里面可能大量还充斥着错误的理解，包括我自己的理解也不一定说就是对的。所以还看各位看官擦亮眼睛才是最重要的。

####  阻塞/非阻塞
![](./java-async-road/blocking_io.png)
发起调用之后，一直等待其数据响应并读取返回。
阻塞：发起调用时不能立即返回，必须需要有结果时才会返回。

![](./java-async-road/noblocking_io.png)
发起调用之后立即返回并通过返回状态码告知调用者数据是否准备返回完毕。
非阻塞：发起调用后立即返回并返回的不一定是最终结果。

#### 所以阻塞/非阻塞IO说的是调用方是否会因IO而blocking，**描述的主体是调用方**。(敲黑板！！！，重点)

下面将举例一个简单的例子。监听端口，一旦有链接建立时，响应hello world。看看两者有什么区别。

```java
//这个是client代码，不重要
public class IOClient {
    public static void main(String[] args) throws Exception {

        SocketChannel client = SocketChannel.open();
        client.configureBlocking(true);
        client.connect(new InetSocketAddress( "localhost", 8080));
        ByteBuffer data = ByteBuffer.allocate(128);
        client.read(data);
        data.flip();
        byte[] arr = new byte[data.remaining()]; //fuck ByteBuffer api
        data.get(arr); //fuck ByteBuffer api
        System.out.println(Arrays.toString(arr));
        System.out.println(new String(arr, StandardCharsets.UTF_8));
    }
}
```

//这是阻塞的，
```java
    public static void main(String[] args) throws Exception {

        SocketChannel socket = getSocket(); //一直阻塞到有连接进来时，即创建socket
        byte[] bytes = "hello world".getBytes(StandardCharsets.UTF_8);
        System.out.println(Arrays.toString(bytes));
        socket.write(ByteBuffer.wrap(bytes)); //
        socket.close();
    }


    public static SocketChannel getSocket() throws Exception {
        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(true);
        ssChannel.bind(new InetSocketAddress(8080));
        System.out.println("bind 8080");
        SocketChannel socket = ssChannel.accept();//blocking io
        socket.configureBlocking(true);
        return socket;
    }
```

//这是非阻塞的
```java
public static void main(String[] args) throws Exception {
        startServer(socket -> {
            byte[] bytes = "hello world".getBytes(StandardCharsets.UTF_8);
            System.out.println(Arrays.toString(bytes));
            try {
                socket.write(ByteBuffer.wrap(bytes));
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }


    public static void startServer(Consumer<SocketChannel> callback) throws Exception {
        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.bind(new InetSocketAddress(8080));
        System.out.println("bind 8080");
        new Thread(() -> {
            try {
                //需要不断的去检测serverSocketChannel
                while (true) {
                    SocketChannel sc = ssChannel.accept();
                    if (sc == null) {
                        Thread.sleep(1000L);
                    } else {
                        callback.accept(sc);
                    }
                }
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
```
阻塞代码：bind，accept之后阻塞等待到链接的到来。然后write data，最后close。
非阻塞代码：bind，循环accept直到accept返回的socket不为空。write data，最后close。 （这里write 有点小瑕疵，不过不影响这个例子）

#### 两者区别很明显，非阻塞IO需要你不断去检测accpet事件（accept换成read，write亦是如此）。如果这个socket量非常大的话就会影响IO效率。所以就有了epoll，高效的去帮助我们检测socket事件。

讲epoll之前，先来看看异步IO。
上面的阻塞/非阻塞可以看出来，都需要我们**手动检查状态**。所以说它们**都是同步**的。

![](./java-async-road/async_io.png)

在这个异步IO里，发起IO调用立即返回，等IO到达时再回调通知你。
#### 这个异步IO跟上面两个有明显区别是**状态通知机制**。
> 同步：是调用方主动去查询状态。
  异步：是被调用方主动通知你。



下面演示一下Java8自带的异步IO代码
```java
public static void main(String[] args) throws IOException, InterruptedException {
        AsynchronousServerSocketChannel ssChannel = AsynchronousServerSocketChannel.open();
        ssChannel.bind(new InetSocketAddress("localhost", 8080));
        System.out.println("listening 8080");
        ssChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel result, Void attachment) {
                byte[] bytes = "hello world".getBytes(StandardCharsets.UTF_8);
                System.out.println(Arrays.toString(bytes));
                result.write(ByteBuffer.wrap(bytes));
                try {
                    result.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });

        Thread.sleep(300000000L);
    }

```
跟上面的非阻塞IO相比，这个使用起来就更加多了，不再需要写while主动去检测状态，而是注册一个回调方法并在回调方法中处理发送data的业务。
同样的，非阻塞IO经过稍加封装之后也可以跟上述的异步IO一样，注册一个回调方法即可。
这里要稍加说明的是这个jdk提供的异步IO（也就是Java NIO2），实际上也是封装IO多路复用实现的。也就是说这个异步IO是假的，通过封装epoll来实现的（特指Linux环境）。同时这个有一个非常明显的缺点就是很难自己精确控制回调执行所在的线程。
关于这个“异步IO”，我会专门出一篇文章来说说。因为一些国产“异步IO”框架，穿上伪异步IO的外衣，就秒天秒地，轻松干掉netty，C10000000000K也不再是难题。


### 小阶段总结：
#### 阻塞/非阻塞是对于调用方角度来描述的，调用方法调用这个方法会不会阻塞。
#### 同步/异步是对于调用方和被调用方通知机制来描述的。
#### 同步：需要调用方手动去检测。异步：被调用方主动通知你，而这个通知通常是callback的方式。
> 所以，同步阻塞IO，同步非阻塞IO，IO多路复用（epoll）都是同步的,这些IO都需要在数据到来之时主动去读取。

对于同步阻塞IO，同步非阻塞IO，异步非阻塞IO在Linux能找到对应的IO模型。
唯独异步阻塞IO找不到一个与之对应的IO模型，主要是这个IO模型没啥实际用途。调用方法阻塞，然后回调通知你？？？ 你想想看也没有对应的应用场景吧。

#### IO多路复用
前面说了，非阻塞IO，需要自己不断的检测所有的socket。这种操作非常低效。所以就有了IO多路复用，其中代表就是epoll。关于epoll的原理，这篇文章讲的挺不错的，所以直接看这个文章就好了（[大话select，poll，epoll](https://cloud.tencent.com/developer/article/100548)）
这里说个它们区别的总结：
- epoll的时间复杂度是O(1)的，而select，poll的时间复杂度是O(n)的。所以理论上，epoll不受连接数影响，select，poll连接越多效率越低。


这里，举一个稍微麻烦一点的例子，来实现基于同步非阻塞IO的异步非阻塞代码。
client连接server之后，发送消息。然后server分3次发1，2，3，再关闭socket。

```java
//server 代码
public class EpollServer {
    public static void main(String[] args) throws Exception {
        startServer(socket -> {
            
            socket.read(result -> {
                if(result instanceof String) { //
                    System.out.println("result -> " + result);
                    
                    socket.write("1", r1 -> {
                        if(r1 == null) { //write ok
                            
                            socket.write("2", r2 -> {
                                if(r2 == null) { //write ok
                                    
                                    socket.write("3", r3 -> {
                                        System.out.println("socket close");
                                        socket.close();
                                    });
                                }
                            });
                        }
                    });
                }
            });
        });
    }

    public static void startServer(Consumer<Socket> consumer) throws Exception {

        Selector selector = Selector.open();
        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);
        ssChannel.bind(new InetSocketAddress(8080));
        System.out.println("bind 8080");
        new Thread(() -> {
            try {
                while (true) {
                    int select = selector.select();
                    if(select <= 0) {
                        Thread.sleep(1000L);
                        continue;
                    }
                    Set<SelectionKey> keys = selector.selectedKeys();
                    Iterator<SelectionKey> iterator = keys.iterator();
                    while (iterator.hasNext()) {
                        SelectionKey key = iterator.next();
                        SelectableChannel channel = key.channel();
                        if(channel instanceof ServerSocketChannel) {
                            ServerSocketChannel serverChannel = (ServerSocketChannel) channel;
                            SocketChannel socketChannel = serverChannel.accept();
                            socketChannel.configureBlocking(false);
                            key = socketChannel.register(selector, 0);
                            Socket socket = new Socket(socketChannel, key);
                            key.attach(socket);
                            consumer.accept(socket);

                        } else if(channel instanceof SocketChannel) {
                            SocketChannel socketChannel = (SocketChannel) channel;
                            Socket socket = (Socket) key.attachment();
                            if((key.readyOps() & SelectionKey.OP_WRITE) != 0) {
                                Consumer<Object> callback = socket.getWriteCallback();
                                socket.setWriteCallback(null);

                                int ops = key.interestOps();
                                ops &= ~SelectionKey.OP_WRITE;
                                key.interestOps(ops);

                                if(socket.getPendingWrite() == null) {
                                    if(callback != null) {
                                        callback.accept(new NullPointerException("without data for write"));
                                    } else {
                                        socket.close();
                                    }
                                } else {
                                    String data = socket.getPendingWrite();
                                    socket.setPendingWrite(null);
                                    socketChannel.write(ByteBuffer.wrap(data.getBytes()));
                                    callback.accept(null);
                                }
                            } else if((key.readyOps() & SelectionKey.OP_READ) != 0) {
                                Consumer<Object> callback = socket.getReadCallback();
                                socket.setReadCallback(null);

                                ByteBuffer data = ByteBuffer.allocate(128);
                                int read = socketChannel.read(data);
                                if(read < 0) { //对端关闭
                                   socket.close();
                                   continue;
                                }
                                data.flip();
                                byte[] arr = new byte[data.remaining()]; //fuck ByteBuffer api
                                data.get(arr);
                                if(callback == null) {
                                    System.out.println("no readCallback");
                                    socket.close();
                                    continue;
                                }
                                callback.accept(new String(arr));
                            }
                            //other op
                        }
                        iterator.remove();
                    }
                }
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }

    static class Socket {
        private final SocketChannel socketChannel;
        private final SelectionKey key;
        private String pendingWrite;
        private Consumer<Object> readCallback;
        private Consumer<Object> writeCallback;

        public Socket(SocketChannel socketChannel, SelectionKey key) {
            this.socketChannel = socketChannel;
            this.key = key;
        }

        public void read(Consumer<Object> callback) {
            key.interestOps(key.interestOps() | SelectionKey.OP_READ);
            this.readCallback = callback;
        }

        public void write(String data, Consumer<Object> callback) {
            key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);
            this.pendingWrite = data;
            this.writeCallback = callback;
        }

        public String getPendingWrite() {
            return pendingWrite;
        }

        public void setPendingWrite(String pendingWrite) {
            this.pendingWrite = pendingWrite;
        }

        public Consumer<Object> getReadCallback() {
            return readCallback;
        }

        public void setReadCallback(Consumer<Object> readCallback) {
            this.readCallback = readCallback;
        }

        public Consumer<Object> getWriteCallback() {
            return writeCallback;
        }

        public void setWriteCallback(Consumer<Object> writeCallback) {
            this.writeCallback = writeCallback;
        }

        public void close() {
            try {
                this.socketChannel.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```
```java
//client代码
public class IOClient {
    public static void main(String[] args) throws Exception {
        SocketChannel client = SocketChannel.open();
        client.configureBlocking(true);
        client.connect(new InetSocketAddress( "localhost", 8080));
        client.write(ByteBuffer.wrap("hello server".getBytes()));
        while (true) {
            ByteBuffer data = ByteBuffer.allocate(128);
            client.read(data);
            data.flip();
            byte[] arr = new byte[data.remaining()]; //fuck ByteBuffer api
            data.get(arr); //fuck ByteBuffer api
            System.out.println(Arrays.toString(arr));
            System.out.println(new String(arr, StandardCharsets.UTF_8));
            if("3".equals(new String(arr))) {
                System.out.println("client close");
                break;
            }
        }
    }
}
```

主要看main方法的代码，一个基于同步非阻塞IO的代码，必然会带来回调问题。
startServer方法中的Thread会一直跑，不会因为IO而陷入阻塞，同时在callback方法里不能执行阻塞代码，否则会影响该线程后续的IO。所以也就为什么Netty，Vert.x，Webflux中不能执行阻塞代码，就是因为阻塞代码导致该任务执行时间太久而影响后续的IO事件。

下层基于epoll来实现非阻塞IO的事件通知机制，上层包装成异步回调通知机制，也正是现阶段Java异步非阻塞框架的实现机制。


在我们日常使用的异步非阻塞框架中，通常经过几层封装之后，已经解决一部分callback hell的问题，导致使用的时候看不太出来是异步的.
例如webflux的注解模式，只需要在声明方法的时候返回Mono，Flux即可。而且里面所有的异步组件都被封装成Reactive模式。
这些异步组件执行到最里面的时候就是IO多路复用组件了，所以**这些方法的调用不会因IO而阻塞**。然后通过Reactive对象封装状态并返回，然后在这个Reactive对象设置回调即可。



### 总结:
1. 所以我们所说的异步非阻塞框架，其实说的是基于同步非阻塞IO实现的异步非阻塞框架。
2. 这些异步非阻塞框架因为底层是非阻塞IO，不会因为IO导致线程阻塞以及线程上下文切换，恢复等一系列工作，且因为每个线程都是无阻塞的高效运行着，所以只需要少量的线程尽可能跑满cpu即可。
3. 因为上层因为异步，下面往往是epoll实现的事件通知机制来处理。所以任何异步方法的调用都伴随着callback。

等真正理解异步之后，这些开源领域的异步框架用法都差不多，其他都是架构上的不同了。而且掌握了一种之后，再看看其他的，轻而易举。
所以对于异步框架来说，理解非常重要，不然非常容易就使用错。特别是在同步，异步混着用的情况下。


最后，什么是异步？
理解了吗？

- 不因IO调用而阻塞,最底层使用的是IO多路复用（epoll）。
- 所有的方法都是非阻塞的，IO线程在最底层高效的运行着。（这也意味着跟Java目前生态的组件大多数冲突的，所以异步组件的生态往往也比较差。）
- 所有的请求和链接不再独占线程，它们只需要占有一点内存即可。
- 根据以上几点，异步可以大大提升吞吐量，从而增强server的可伸缩性。


为什么同步性能和吞吐量远不如异步？
- 所有的IO调用都是阻塞的，因为阻塞，所以需要大量的线程，而大量的线程会导致大量的线程挂起，恢复，上下文切换等问题。导致业务代码等有效作业做工比值低。


期待project Loom的到来，这对于Java来说，可能是一个革命性的新特性。

--- 
由于篇幅有限，这篇文章暂时先到这里。前面提到了callback，居然callback是个问题，那么接下来就说说如何解决callback hell的问题。

参考链接：
- [IO - 同步，异步，阻塞，非阻塞](https://blog.csdn.net/historyasamirror/article/details/5778378)
- [大话select，poll，epoll](https://cloud.tencent.com/developer/article/100548)


--- 
好了，晚安。



















































           





















 



















































