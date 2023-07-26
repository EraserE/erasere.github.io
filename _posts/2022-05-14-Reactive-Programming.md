---
title: Reactive Programming
date: 2022-05-14 18:30:00 -0500
categories: [笔记]
tags: [Reactive Programming]
pin: true
author: Eraser

toc: true
comments: true
typora-root-url: ../../eraseryao.github.io
math: false
mermaid: true
---

# Reactive Streams

## 基本概念

Java9开始，通过Reactive Streams和Flow API实现响应式编程。

### Stream-Oriented Pub/Sub patterns

使用Pub-Sub模型，publisher负责发送消息，subscriber负责通知publisher自己可以接受多少消息并接受。

包含两个部分

**Iterator**: which applies a “pull model” where app subscriber(s) pull items from a publisher source.

**Observer**: which applies a “push model” that reacts when a publisher source pushes an item to subscriber sink(s).

### Flow API

Flow API包含三个组成部分：

Publisher相当于信息的发布源，Subscriber负责消化Publisher发布的事件；

Publisher通过钩子函数onNext，onError, onComplete向Subscriber发送信息；

Subscription用来控制Pub和Sub之间的数据流。

![截屏2023-05-14 20.57.19](/assets/blog_res/2022-05-14-Reactive-Programming.assets/%E6%88%AA%E5%B1%8F2023-05-14%2020.57.19.png)

由于Reactive Streams只有当.subscribe()方法被调用时才会执行，所以是lazy的。

当Subscriber发送请求给Publisher后，Publisher会调用onSubscribe钩子函数，可以确保Sub可以接受到信息；

Publisher调用onNext(data)钩子函数来回复请求；

如果事件结束，Pub会调用onComplete钩子函数。



