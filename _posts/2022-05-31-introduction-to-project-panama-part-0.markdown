---
layout: post
title:  'Introduction to OpenJDK Project Panama. Part 0: "Hello Project Panama" application.'
date:   2022-05-31
categories: openjdk panama
tags: ["openjdk", "panama"]
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Intro

JDK 19 includes JEP 424, Foreign Memory Access & Function Interface API, as a preview feature. This marks the first major Project Panama feature to move out of incubation and into preview feature status. With the release of JDK 19 less than four months away, it's time to talk about Project Panama. 

This article series serves as an introduction to Project Panama. We will cover; essential concepts, techniques, and methodologies, core APIs, and how developers can start taking advantage of Project Panama APIs now. 

If you are already familiar with Project Panama and want to start exploring its APIs, check out the following article in the series where we build a Hello World application C-style in Java.   

## What is Project Panama?

Project Panama serves as the interface between two worlds: the JVM and native code written in other languages like C/C++. 

> **BILLY NOTE:** A bit more depth here, how does panama improve on JNI? Why is it being done?

Project Panama consists of three main components:

  * Foreign Function & Memory API: [JEP 424](https://openjdk.java.net/jeps/424)
  * Vector API:  [JEP 338](https://openjdk.java.net/jeps/338)
  * [Jextract tool](https://github.com/openjdk/jextract):  

> **BILLY NOTE:** Provide a couple-sentence explanation for each of the items in the list above. 


## Where to start?

First, you will need the latest [OpenJDK early access build](https://jdk.java.net/19/). It will have the most up-to-date changes to the APIs in Project Panama we will be covering in this article series. 

## What you will learn

You will get your hands dirty working with the Project Panama APIs during this article series. The key APIs we will be working with are:

* [Linker](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Linker.html) and a [symbol lookup](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SymbolLookup.html) - a set of API classes to perform upcalls and downcalls.
* [Memory layout](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryLayout.html) and [descriptors](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html) - APIs to model foreign types (structures, primitives, function descriptors). 
* [Memory segment](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySegment.html) and [its address](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryAddress.html) - a set of API classes to work with native memory and pointers.
* [Memory session](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySession.html) - an abstraction to manage the lifecycle of one or more memory resources.
* [Segment allocator](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SegmentAllocator.html) - an API to allocate memory segments within a memory session.

## Series outline

This series will consist of # articles. 

Part 1 - We will build a HelloWorld application C-style in Java. We will learn about Linker and SymbolLookup APIs and touch on native memory management using this example. [Link]()

Part 2 - 

## Summary 

In this article, we did a high-level overview of Project Panama. 

> **BILLY NOTE:** Why should developers be interested in this series? How might all developers benefit from learning about Panama.

If you are ready to start getting your hands dirty with Project Panama, check out the next article in the series where we write a HelloWorld application C-style in Java. [Link]()

