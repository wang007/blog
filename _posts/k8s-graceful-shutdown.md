---
title: 在 k8s 环境中如何优雅关闭 server
date: 2020-11-01 21:35:52
tags:
  - java
  - k8s
  - 优雅关闭
  - graceful shutdown
  
categories:
  - java
  - k8s
---

最近在线上环境个别服务更新时，服务调用方出现 "Connection reset by peer"，经过最终排查发现服务端没有进行 graceful down
先交代一下环境：
1. 部署平台: k8s
2. 服务调用: grpc
3. spring boot
4. 服务调用 client 是基于 grpc client 封装的，也仅仅是处理了连接管理。服务发现基于 k8s service 

<!-- more -->

当个别服务更新时，服务调用方出现 "Connection reset by peer" 异常，对应线上环境这种服务更新导致的调用问题是不可接受的，这会影响系统的稳定性。

## 1. Connection reset by peer 初步分析
实际上这是一个很常用的异常，熟悉 tcp 的话应该很清楚。
这种情况发现在服务端突然把连接关闭了，客户端不知道，然后继续往服务发送数据，服务端就会返回 rst 包给客户端，最终造成这种现象。

也就是说，服务端没有进行优雅关闭，去告知到客户端要关闭连接了。但是通过查看服务部署文件实际上和服务端代码，实际上有进行优雅关闭的。

## 2. client 端初步分析
服务端看起来没问题。然后异常又是 client 端报的，而且最重要的是 grpc-client 刚更新一版，让人不得不怀疑到 client 端。
深入分析了几天的 grpc 源码，并没有发现异常。而且通过分析源码来解决 bug 的效率非常低下。不得不转头继续分析 server 端。
  
## 3. 分析 k8s 关闭 pod 流程和 server 端 shutdown 代码
由于分析源码没有进展而且一进到源码里细节很多，非常耗时间。
通过分析 k8s 关闭 pod 流程和 server 端 shutdown 代码，果然发现了蛛丝马迹。
  
###  k8s 关闭 pod 流程
当 k8s 关闭 pod 时，会把 pod 标记为 "Terminating" 状态和记录一些删除相关的信息。 kubelet 会收到 pod "Terminating"信息，kubelet 开始优雅关闭 pod。
  1. 先执行 pod preStop 配置的命令（如果有配置命令的情况下）。
  2. 在 preStop 执行完时，pod 就会收到 TERM 信号。尝试关闭进程。
  3. 上述过程是串行执行的，也就是 2 等待 1 执行完。
  4. 这整个过程称为 graceful shutdown 过程。 k8s 默认等待 graceful shutdown 为 30s，可配置。
  
  与此同时，会跟此过程 **并行**  **执行 pod 对应 endpoint 的删除**，也就是说在 k8s service LB 中摘除该 pod。
  但是往往 "删除endpoint 时间" > "删除 pod 时间"。 就是说 LB 中还存在该 pod 路由信息，但是 pod 已经关闭了。
  这时 client 端就有可能出现 "Connection refused"， http 502 状态码之类的错误。
  
  所以为了先正确的摘除流量，一般会在 pod preStop sleep 一段时间。而这个服务就是设置 sleep 30s
  也就是说为了 "删除 endpoint 时间" < "删除 pod 时间"，即service LB 过来到该 pod 流量还能正常处理，一旦 endpoint 删除时，新流量就不会到该 pod 导流量了。 
  sleep 的时间就是等待 "删除 endpoint 时间"，而且这个时间设置很关键。设大设小都不好。设小了跟没设置的效果差不多，设置大了会延长服务部署时间。
  
  前面说了 graceful shutdown 过程 k8s 默认会等待 30s， 也就是说 preStop sleep 30s 的话，那么给进程关闭的时间就没了，
  导致进程根本没有时间执行 graceful shutdown。虽然 k8s 再给 2s 进程进行关闭，但是 2s 太小了。
  所以**当 preStop sleep 设置比较大的值时，一定要注意给到进程的关闭时间。** 可通过参数 pod terminationGracePeriodSeconds 延长 k8s 等待 graceful shutdown 时间。
  更详细资料参考 [Pod 的生命周期](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
  
  
### server 端 shutdown 代码
当 java 服务端需要优雅关闭时，最原始的方式可以往 Runtime 类里 register shutdown hook，例如
```java
    public class Main {
        public static void main(String[] args) {
            Runtime.getRuntime().addShutdownHook(new Thread(() -> {
                System.out.println("exec on shutdown hook");
            }));
        }
    }
```

但是一般情况下，我们用的是 spring。那么 spring 有多种方式来执行 shutdown hook， 例如：
```java
     @Component
     public class Main implements DisposableBean {
         @Override
         public void destroy() throws Exception {
             System.out.println("exec on shutdown hook");
         }
     }

    @Component
    public class PreStopTest {
    
        @PreDestroy
        public void stop() {
            System.out.println("exec on shutdown hook");
        }
    }

    //或者监听 ContextClosedEvent 等等方式
```

spring 的实现最终也是往 Runtime register shutdown hook 的，具体代码
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
        
    public void registerShutdownHook() {
		    if (this.shutdownHook == null) {
		    	// No shutdown hook registered yet.
		    	this.shutdownHook = new Thread() {
		    		@Override
		    		public void run() {
		    			synchronized (startupShutdownMonitor) {
		    				doClose();
		    			}
		    		}
		    	};
		    	Runtime.getRuntime().addShutdownHook(this.shutdownHook);
		    }
	}
}

```

shutdownHook 线程里中的 doClose 方法就是执行各种 shutdown 操作，包括前面提到 DisposableBean 
@PreDestroy ContextClosedEvent 等。

而问题往往会出现优雅关闭的处理上，例如为了 servlet 容器能安全的关闭，往往会等待 servlet 容器关闭，那么阻塞等待就会导致后面的 bean 没机会执行 shutdown。
这次的 bug 主要就是因为 servlet 容器等待阻塞了 30s，导致 grpc server 没机会执行 shutdown。


### 4. 修复
根据前面的分析，引起这次 bug 的主要有两个。
1. pod preStop sleep 时间的设置
   这个跟运维商量之后，这个值是经过生产可靠的值。那么就不修改这个值，而是修改 pod terminationGracePeriodSeconds = 50s，
   把 k8s 等待 graceful shutdown 时间改成50s，相当于给进程 20s 的关闭时间。

2. servlet 容器阻塞等待导致后面的 bean 没机会执行
   像 servlet 容器，grpc server 这种比较大资源的关闭且需要阻塞等待的，通过直接往 Runtime register 
   shutdown hook的方式，这种就不会影响其他 bean 执行 shutdown。 例如
   
```java  
        Thread jettyShutdownHook = new Thread(() -> {
            // do something, 例如等待阻塞时间计算
             try {
                  server.stop();
                  log.info("Jetty server stopped.");
             } catch (Exception e) {
                  log.error("Fail to shutdown gracefully.", e);
             }
        });
        jettyShutdownHook.setDaemon(false);
        jettyShutdownHook.setName("jetty-shutdown-hook-thread");
        Runtime.getRuntime().addShutdownHook(jettyShutdownHook);
```

```java
    Thread grpcShutdownHook = new Thread(() -> {
         try {
              this.destroy();
         } catch (Exception e) {
              log.error("", e);
         }
    });
    grpcShutdownHook.setName("grpc-shutdown-hook-thread");
    grpcShutdownHook.setDaemon(false);
    Runtime.getRuntime().addShutdownHook(grpcShutdownHook);
```

## 总结
1. 平时我们写代码往往会忽略掉优雅关闭的问题，但是这些不起眼的小细节反而可能导致一些难以排查的 
bug。所以平时写代码要注意优雅关闭的情况和资源释放的问题。

2. 优雅停机的第一步就是停流量，在 k8s 环境里就是等待 endpoint 和 删除 endpoint 过程流量导到即将删除 
pod 时，pod 还能正常处理。 所以一般通过 pod preStop sleep 一段时间的方式解决，在 sleep 时期，pod 还是正常运行的，即能正常处理流量。   

3. 需要在 shutdown 时，阻塞等待的 bean，可以通过直接往 Runtime register shutdown hook 方式来解决（当然其他方式亦可），不阻塞 spring 其他 bean 执行。
   



