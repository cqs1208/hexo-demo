---
layout: post
title: 02 FutureTask源码解析
tags:
- JavaCore
categories: JavaCore
description: JavaCore
---

Future是Java执行异步任务时的常用接口

<!-- more --> 

## 1. 背景与简介

Future是Java执行异步任务时的常用接口。我们通常会往ExecutorService中提交一个Callable/Runnable并得到一个Future对象，Future对象表示异步计算的结果，支持获取结果，取消计算等操作。在Java提供的Executor框架中，Future的默认实现为java.util.concurrent.![futureTask](/images/JavaCore/JavaCore_futureTask.png)

可以看到,FutureTask实现了RunnableFuture, 而RunnableFuture的JavaDoc对Runnable接口的run方法有了更精确的描述：run方法将该Future设置为计算的结果，除非计算被取消。 

## 2. 源码解读

### 2.1 生命周期状态

FutureTask内置一个被volatile修饰的state变量。 按照生命周期的阶段可以分为： 

- NEW 初始状态
- COMPLETING 任务已经执行完(正常或者异常)，准备赋值结果
- NORMAL 任务已经**正常**执行完，并已将任务返回值赋值到结果
- EXCEPTIONAL 任务执行失败，并将异常赋值到结果
- CANCELLED 取消
- INTERRUPTING 准备尝试中断执行任务的线程
- INTERRUPTED 对执行任务的线程进行中断(未必中断到)

### 2.2 内部结构

1. Callable callable
   内部封装的Callable对象。如果通过构造函数传的是Runnable对象，FutureTask会通过调用Executors#callable，把Runnable对象封装成一个callable。
2. Object outcome
   用于保存计算结果或者异常信息。
3. volatile Thread runner
   用来运行callable的线程。
4. volatile WaitNode waiters
   FutureTask中用了[Trieber Stack](https://en.wikipedia.org/wiki/Treiber_Stack)来保存等待的线程。

