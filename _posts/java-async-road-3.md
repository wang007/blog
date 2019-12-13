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

相信你如果是第一次接触它的话，很懵逼。到底是干啥的啊，有什么用啊。我当初也是一样。就像当初学异步一样，什么是异步啊。

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
> 首先的是异步，得在异步环境才有使用响应式的必要。同步不是说不行，同步直接返回raw对象就行了，没必要再用ReactiveX对象去包装了。

#### 2.变化和传播
> 响应式提供了丰富的操作符对响应式对象进行变换。例如 map，flatMap，filter等等。

#### 3. 编程范式
> 这个可以类比数据库范式，数据库范式就很熟悉了吧，它可以帮助更有效的组织数据且设计表的一个参考。但是我们往往有时候为了提高查询效率，往往会做一些适当的数据冗余。所以说范式只是一个参考，一种规范吧，是充分非必要的。编程范式也是类似的，遵守这个范式可以让代码的可阅读性大大增强。callback hell的可阅读性就非常的差。

实际上按照响应式的定义，CompletableFuture也是响应式对象。也就是说使用CompletableFuture编程的话也可以叫做响应式编程。只不过CompletableFuture的数据流是单一的，只能发射一个对象。
跟Rxjava#Single，ProjectReactor#Mono类似。

### 正文
先来个直观的感受，本系列第一篇文章有段回调嵌套的代码。现在用响应式编程的方式解决掉回调的问题。











