---
layout: post
title:  'Introduction to OpenJDK Project Panama. Part 1: "Hello World" application.'
date:   2022-05-31
categories: openjdk panama
tags: ["openjdk", "panama"]
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Introduction

With JDK 19 getting released in the coming weeks, it's time to talk about Project Panama, 
which APIs are going to be available for developers and how can developers use it even now.

In this article, I will guide you through what every software developer dealing with 
when learning a new programming language - a "hello world" application, a C version of "hello world" written in Java!

### Prerequisites

First, you would need the latest [OpenJDK early access build](https://jdk.java.net/19/) (build 24 or greater).
Please visit Java developer portal [dev.java](https://dev.java/learn/getting-started-with-java/#anchor_5) to learn how to install and configure the JDK.


## OpenJDK Project Panama: what you need to know?

[OpenJDK Project Panama](https://openjdk.java.net/projects/panama/) is meant to be a bridge between two worlds: the JVM and native code written in other languages like C/C++.

So, Panama consists of 3 components:
* Foreign Function & Memory API: [JEP 424](https://openjdk.java.net/jeps/424)
* Vector API: [JEP 338](https://openjdk.java.net/jeps/338)
* [Jextract tool](https://github.com/openjdk/jextract)

When you bury your hand into these APIs you will need to operate with a couple of abstraction:
* [Memory segment](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySegment.html) and [its address](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryAddress.html) - a set of API classes to work with native memory and pointer to it;
* [Memory layout](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryLayout.html) and [descriptors](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html) - APIs to model foreign types (structures, primitives, function descriptors).
* [Memory session](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySession.html) - an abstraction to manage the lifecycle of one or more memory resources;
* [Linker](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Linker.html) and a [symbol lookup](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SymbolLookup.html) - a set of API classes to perform down- and upcalls.
* [Segment allocator](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SegmentAllocator.html) - an API to allocate memory segments within a memory session;

## Hello World!

As the further you dig, the more you see that to start working with Panama properly, it's so critical to have a good introduction
just to make sure you haven't missed important concepts, techniques, and methodologies.

So, in Part 1, I will only cover **Linker**, also briefly address **SymbolLookup** methods and native memory management.
These three major components of Panama are building blocks for more in-depth development of programs consisting of Java and native code.

## Linker

From a technical standpoint, we’re talking about a _“bridge”_ between two binary interfaces: the JVM and C/C++ native code,
also known as [C ABI](https://en.wikipedia.org/wiki/Application_binary_interface).

Developers often work with C/C++ native code in a format of shared libraries of a platform-specific format (`*.so, *.dylib, *.dll`) and their header files.
Whenever you start a ordinary hello-world Java application, the JVM talks to C standard library to execute one of many C functions like _printf_.

The JDK 19 offers a set of the C ABI implementation for all popular platforms (including latest macOS AARCH64):
```java
public static Linker getSystemLinker() {
    return switch (CABI.current()) {
    case Win64 -> Windowsx64Linker.getInstance();
    case SysV -> SysVx64Linker.getInstance();
    case LinuxAArch64 -> LinuxAArch64Linker.getInstance();
    case MacOsAArch64 -> MacOsAArch64Linker.getInstance();
    };
}
```

In the JDK terminology, [**Linker**]((https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Linker.html)) is an instance of the platform-specific C ABI implementation.
Linker offers a set of API methods designed to perform both downcalls and upcalls, where:
* a _downcall_ is an event initiated from a high-level subsystem,
  in our case the JVM to a lower-level subsystem, like the OS kernel. I'll explain it later in the text in the context of Foreign Function & Memory API.
* an _upcall_. The opposite of a downcall.

While the **Linker** is like your telephone - call whoever you want to, just dial in a proper phone number.
The symbol lookup methods are like your phonebook - just provide a proper descriptor of who you want to call!

To perform a downcall, you need a descriptor of a function you’d like to call, a native address allocated through a symbol lookup,
and a linker to create a method handle that you can invoke.

Implementing a classic C-style Hello World from Java:
```cpp
int     printf(const char * __restrict, ...)
```

## "Hello World" C-style in Java

This section will cover what has to be done in a form of tasks followed one by another in a logical sense.

### 1. Find the address of the native function.

We need to do couple things, first, to look up for a native memory address of the _printf_ function:
```java
SymbolLookup linkerLookup = linker.defaultLookup();
SymbolLookup systemLookup = SymbolLookup.loaderLookup();

SymbolLookup symbolLookup = name ->
        systemLookup.lookup(name).or(() -> linkerLookup.lookup(name));

Optional<MemorySegment> printfMemorySegment = symbolLookup.lookup("printf");
```

Since [_printf_](https://www.cplusplus.com/reference/cstdio/printf/) is a part of C stdlib, you don't have to expect that a lookup will fail.
However, a lookup may fail due to multiple reasons, it will be cover it in the upcoming posts.

### 2. Build a descriptor of a function you are calling.

Once we know where C _printf_ resides, the second thing to do is to define the _printf_ descriptor that consists of a result type and accepted parameters.
In Java, a method that accept a variables set of parameters is called a varargs method, the equivalent in C (ex. _printf_) is called a variadic function.

Note: from a Java runtime standpoint, it doesn't matter what value type stands behind the C pointer because a memory layout for a C pointer does not hold type but a 32/64-bit value fixed by the platform.

To simplify Part 1, I'll define a simplified version of **FunctionDescriptor** for _printf_:
```java
FunctionDescriptor printfDescriptor = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
    );
```

A descriptor defines a function which return value is _int_ and a parameters is a pointer.
It _almost_ corresponds to its C definition from [stdio.h](https://www.cplusplus.com/reference/cstdio/printf/), 
because given descriptor defines a non-variadic function which _printf_ definitely is not. 
The upcoming Part 2 will explain how to implement C variadic functions in Java. 

The documentation of [**FunctionDescriptor**](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html#of(java.lang.foreign.MemoryLayout,java.lang.foreign.MemoryLayout...)) declares the following signature:
```java
FunctionDescriptor of(MemoryLayout resLayout, MemoryLayout... argLayouts)
```

a method **FunctionDescriptor::of** accepts a memory layout as both return value type and a function's arguments,
in case of simplified _printf_ - _int_ layout as a return value and a pointer as named argument, however, 
ideally, it has to be a pointer and varargs, but it also could be whatever a function needs.

#### Modeling C types in Java through value layouts

In Java, A value layout is used to model the memory layout associated with values of basic data types, 
such as integral types (either _signed_ or _unsigned_) and floating-point types. Both _JAVA_INT_ and _ADDRESS_ are value layouts of corresponding C types.

[JAVA_INT](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/ValueLayout.OfInt.html):
```java
// ValueLayout.OfInt.class
OfInt JAVA_INT = new OfInt(ByteOrder.nativeOrder()).withBitAlignment(32);
```
is an instance of a value layout whose carrier is **int.class**.
With this layout the **Linker** is instructed to create a bridge between C _int32_
and a corresponding Java _int_ type with a carrier-class **int.class**.

[ADDRESS](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/ValueLayout.OfAddress.html):
```java
// ValueLayout.OfAddress.class
OfAddress ADDRESS = new OfAddress(ByteOrder.nativeOrder())
            .withBitAlignment(ValueLayout.ADDRESS_SIZE_BITS);
```
is a value layout which corresponding C type is a pointer to a variable, a carrier is **MemoryAddress.class**.

### 3. Build a method handle from a function's native memory address.

Using the printf native address and its Java (function?) descriptor, we can now create a method handle for C _printf_:
```java
MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
            addr -> linker.downcallHandle(addr, printfDescriptor)
    ).orElse(null);
```
Given code create the _printf_ a directly executable reference of an underlying method, in short - a method handle,
out of a native memory segment of _printf_ and its simplified descriptor.
Upon the unsuccessful lookup, we potentially will get a null method handle which would be a signal of an issue.

With all necessary concepts explained, it would be better to extend both definitions of down- and upcalls:
* a _downcall_ is an invocation of a native function through a **MethodHandle** formed from a native function address and its Java version of a function descriptor;
* an _upcall_ is an invocation of some code written in Java through a **MethodHandle** converted into a native memory 
segment which could then be passed to native functions as a function pointer.


### 4. Allocate native memory.

Okay, at this stage you may think we have pretty much everything we need to perform the actual downcall, but that's not all.
We need somehow to bind Java objects to native memory segments to make sure C _printf_ can access them.

Both memory allocation and freeing were painful because no matter how good software developer you are, eventually you will forget to allocate or free a memory which will cause a
program to leak it or blow up with a segmentation fault.

Java relies on a Garbage Collector to allocate and free memory but Panama’s FMA API allocates memory off-heap.
In Panama, the FMA API helps to allocate a memory off-heap which is a crucial part of any native interop story!

The FMA API allows to allocate and access memory segments, their addresses, and the shape of contiguous memory
regions, located either on- or off- the Java heap.
All allocated memory segments are bound to a specific memory session (**MemorySession**), 
an instance of it offers a set of APIs to allocate memory segments. Consider a memory session like a unified memory allocation tool,
like C `malloc`.
One of the important features of **MemorySession** is an implementation of an **AutoClosable** interface - 
it is better to use it within the try-with-resources syntax code block.

Last, but not least important part of any program performing downcalls is an instance of **SegmentAllocator**:
```java
try (var memorySession = MemorySession.openConfined()) {
    // memory allocation happening here
    SegmentAllocator allocator = SegmentAllocator.newNativeArena(memorySession);
}
```

There are more than one right way to allocate a memory segment, 
one of the possible memory allocation methods in Java is **SegmentAllocator**, which is similar to **MemorySession**,
but often used in the context of other C memory allocation functions different from C [malloc](https://en.cppreference.com/w/c/memory/malloc).
For the sake of simplicity, this hello world will use **MemorySession** as a memory segment allocation tool.

Finally, to call C _printf_ we need to allocate _const char *_ memory segment within a memory session using **MemorySession** and pass it to C _printf_ function:
```java
MemorySegment cString = memorySession.allocateUtf8String(str + "\n");
```

With the allocated memory segment, we can invoke a function:
```java
private static int printf(String str, MemorySession memorySession) throws Throwable {
    Objects.requireNonNull(printfMethodHandle);
    var cString = memorySession.allocateUtf8String(str + "\n");
    return (int) printfMethodHandle.invoke(cString);
}

public static void main(String[] args) throws Throwable {
    var str = "hello world";
    try (var memorySession = MemorySession.openConfined()) {
        System.out.println(printf(str, memorySession));
    }
}
```

Note that we are allowed to cast a return type to _int_, because a return value layout carrier type of C _printf_ is _int.class_.

### 5. Get things together!

To summarise, invoking a C function from Java:
* Memory session (**MemorySession**) or a segment allocator (**SegmentAllocator**) are key APIs to perform memory allocation;
* Memory session should be declared with a try-with-resources to be automatically closed;
* There are several options to allocate memory segments - through an allocator or a session directly;
* Linker, symbol lookup objects, value/memory layouts, and method handles are static objects.


The full code listing can be found below!

## Summary

The goal of this article is two folds, give an overview of the FMA API by looking at how to invoke a simple C function from Java. 
Achieving such a trivial example might seem a bit complicated but seeing the different steps is key to understand how the FMA API works. 
The good news is that developers can rely on the [jextract](https://github.com/openjdk/jextract) tool to handle most of the FMA machinery. 
Jextract will be covered in an upcoming article.

Several points need to be tackled when invoking native code form Java suing the FMA API:
* Get the native library and its corresponding header files;
* Build a function descriptor in Java (**FunctionDescriptor**);
* Lookup native memory address of a function symbol, and create a method handle for it;
* Create a related method handle and confirm that it has been properly created (ex. if the native libray is not in the system path, 
the lookup will fail and the return a method handle will be null);
* Decide how the application will allocate memory segments: through a segment allocator or a memory session, make sure the memory allocation technique is consistent throughout through the codebase of an application.

## Code listing

You may find sources for this chapter here [github.com/denismakogon/openjdk-project-samples](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-1).

```java
package com.openjdk.samples.panama.stdlib;

import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;
import java.util.Objects;

import static java.lang.foreign.ValueLayout.ADDRESS;
import static java.lang.foreign.ValueLayout.JAVA_INT;


public class Examples {
  private static final Linker linker = Linker.nativeLinker();
  private static final SymbolLookup linkerLookup = linker.defaultLookup();
  private static final SymbolLookup systemLookup = SymbolLookup.loaderLookup();
  private static final SymbolLookup symbolLookup = name ->
          systemLookup.lookup(name).or(() -> linkerLookup.lookup(name));
  private static final FunctionDescriptor printfDescriptor = FunctionDescriptor.of(
          JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
  );

  private static final MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
          addr -> linker.downcallHandle(addr, printfDescriptor)
  ).orElse(null);

  private static int printf(String str, MemorySession memorySession) throws Throwable {
    Objects.requireNonNull(printfMethodHandle);
    var cString = memorySession.allocateUtf8String(str + "\n");
    return (int) printfMethodHandle.invoke(cString);
  }

  public static void main(String[] args) throws Throwable {
    var str = "hello world";
    try (var memorySession = MemorySession.openConfined()) {
      System.out.println(printf(str, memorySession));
    }
  }
  
}
```
