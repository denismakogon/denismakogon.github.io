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

With JDK 19 getting released in the coming weeks, it's time to talk about Project Panama, and more specifically about
the new Foreign Function & Memory API (FFA API) which eases interoperability between Java and native code.

This article introduces the Foreign Function & Memory API using a simple Java-based Hello World invoking some C native code.

### Prerequisites

To use the Foreign Function & Memory API and the samples code, make sure to download [JDK 19](https://jdk.java.net/19/) (build 24 or greater).

## Project Panama Overview

[Project Panama](https://openjdk.java.net/projects/panama/) is meant to be a bridge between two worlds: the JVM and native code written in other languages like C/C++.

So, Panama consists of 3 components:
* The Foreign Function & Memory API: [JEP 424](https://openjdk.java.net/jeps/424)
* the [Jextract tool](https://github.com/openjdk/jextract)
* The Vector API: [JEP 338](https://openjdk.java.net/jeps/338)

The Foreign Function & Memory API uses some key abstractions:
* [Memory segment](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySegment.html) and [its address](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryAddress.html) - a set of API classes to work with native memory and pointer to it;
* [Memory layout](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryLayout.html) and [descriptors](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html) - APIs to model foreign types (structures, primitives) and function descriptors.
* [Memory session](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySession.html) - an abstraction to manage the lifecycle of one or more memory resources;
* [Linker](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Linker.html) and a [symbol lookup](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SymbolLookup.html) - a set of API classes to perform down- and upcalls.
* [Segment allocator](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SegmentAllocator.html) - an API to allocate memory segments within a memory session;

Note: the Vector API is not a part of an article series.

## Hello World!

As the further you dig, the more you see that to start working with Panama properly, it's so critical to have a good introduction
just to make sure you haven't missed important concepts, techniques, and methodologies.

This initial article will only cover **Linker**, and also briefly address **SymbolLookup** methods and native memory management (**MemorySession**).
These three major components described above are building blocks for more in-depth development of programs consisting of Java and native code.

## Linker

From a technical standpoint, a **Linker** is a bridge between two binary interfaces: the JVM and C/C++ native code,
also known as [C ABI](https://en.wikipedia.org/wiki/Application_binary_interface).

The JDK 19 offers a set of the C ABI implementation for all popular platforms:
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

In the JDK terminology, a [**Linker**]((https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Linker.html)) is an instance of the platform-specific C ABI implementation.
A Linker provides a set of methods to perform both downcalls and upcalls, where:
* a _downcall_ is an event initiated from a high-level subsystem,
  in our case the JVM to a lower-level subsystem, like the OS kernel, or some Java code invoking some native code. This will be illustrated later with the Foreign Function & Memory API.
* an _upcall_, ex. some native code invoking some Java code.

While the **Linker** is like your telephone - call whoever you want to, just dial in a proper phone number.
The symbol lookup methods are like your phonebook - just provide a proper descriptor of who you want to call!

To perform a downcall, we need a descriptor of the (native) function to call, a native address allocated through a symbol lookup,
and a linker to create a method handle to invoke the native function.

Implementing a classic C-style Hello World from Java:
```cpp
int     printf(const char * __restrict, ...)
```

## "Hello World" C-style in Java

To write a Java-based Hello World that uses native printf functions, we need to:

### 1. Find the address of the native function.

First, we need to lookup the native memory address of the [_printf_](https://www.cplusplus.com/reference/cstdio/printf/) function:
```java
Linker linker = Linker.nativeLinker();
SymbolLookup linkerLookup = linker.defaultLookup();
SymbolLookup systemLookup = SymbolLookup.loaderLookup();

SymbolLookup symbolLookup = name ->
        systemLookup.lookup(name).or(() -> linkerLookup.lookup(name));

Optional<MemorySegment> printfMemorySegment = symbolLookup.lookup("printf");
```

Technically, a lookup may fail, properly handling such failure will be covered in an upcoming article.

### 2. Build a descriptor of a function you are calling.

Once we know where _C printf_ resides, we need to define the _printf_ descriptor that consists of a result type and accepted parameters.
It's worth to mention that native functions like _printf_ called as variadic functions.
In Java, a method that accepts variables set of parameters is called a method with varargs.

To simplify, we can define a simplified version of **FunctionDescriptor** for _printf_:
```java
FunctionDescriptor printfDescriptor = FunctionDescriptor.of(JAVA_INT, ADDRESS);
```

Note: from a Java runtime standpoint, it doesn't matter what value type stands behind a C pointer because a memory layout of a C pointer does not hold the type but a 32/64-bit value fixed by the platform.

A descriptor defines a function whose return value is _int_, and its parameter is a pointer.
Given descriptor _almost_ corresponds to its C definition from [stdio.h](https://www.cplusplus.com/reference/cstdio/printf/),
because it defines a standard function while _printf_ is a variadic function.
The upcoming Part 2 will explain how to implement C variadic functions in Java.


#### Modeling C types in Java through value layouts

In Java, a value layout is used to model the memory layout associated with values of basic data types,
such as integral types (either _signed_ or _unsigned_) and floating-point types.
Both _JAVA_INT_ and _ADDRESS_ are value layouts of corresponding C types.

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
is a value layout in which the corresponding C type is a pointer to a variable, a carrier is **MemoryAddress.class**.

### 3. Build the method handle from the function's native memory address.

Using the C _printf_ native address and its function descriptor, we can now create a method handle for C _printf_:
```java
MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
            addr -> linker.downcallHandle(addr, printfDescriptor)
    ).orElse(null);
```
Given code creates an executable reference of C _print_, in short - a method handle,
out of a native memory address of _printf_ and its function descriptor.

Note: a [method handle](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodHandle.html) is a typed, directly executable reference to an underlying method, constructor,
field, or similar low-level operation, with optional transformations of arguments or return values.

With all necessary concepts explained, it would be better to extend both definitions of down- and upcalls:
* a _downcall_ is an invocation of a native function through a **MethodHandle** formed from a native function address and its Java version of a function descriptor;
* an _upcall_ is an invocation of some code written in Java through a **MethodHandle** converted into a native memory
  the segment which could then be passed to native functions as a function pointer.


### 4. Allocate native memory.

We need somehow to bind Java objects to native memory segments to make sure C _printf_ can access them.

Both memory allocation and freeing in C were painful because no matter how good a software developer you are, eventually
you will forget to allocate or free a memory which will cause a program to leak it or blow up with a segmentation fault.

On the other hand, Java relies on a Garbage Collector to allocate and free memory but Panama’s Foreign Function & Memory API allocates memory off-heap.
In Panama, the Foreign Function & Memory API helps to allocate a memory off-heap which is a crucial part of any native interop story!

The Foreign Function & Memory API allows to allocate and access memory segments, their addresses, and the shape of contiguous memory
regions, located either on- or off- the Java heap.
All allocated memory segments are bound to a specific memory session (**MemorySession**). An instance of a memory session offers a set of APIs to allocate native memory segments.
Consider a memory session like a unified memory allocation tool, like C `malloc`.
MemorySession implements the AutoClosable interface which greatly simplifies de-allocation using the try-with-resources construct.

The Foreign Function & Memory API offer more than one right way to allocate a memory segment,
one of the possible native memory allocation methods is **SegmentAllocator** which is similar to **MemorySession**:
```java
try (var memorySession = MemorySession.openConfined()) {
    SegmentAllocator allocator = SegmentAllocator.newNativeArena(memorySession);
    var cStringFromAllocator = allocator.allocateUtf8String("Hello World" + "\n");
    var cStringFromSession = memorySession.allocateUtf8String("Hello World" + "\n");
}
```
For the sake of simplicity, this Hello World will use **MemorySession** as a memory segment allocation tool.

Finally, to call C _printf_ we need to allocate _const char *_ memory segment within a memory session using **MemorySession** and pass it to C _printf_ function:
```java
MemorySegment cString = memorySession.allocateUtf8String(str + "\n");
```

With the allocated memory segment, we can invoke the function:
```java
private static int printf(String str, MemorySession memorySession) throws Throwable {
    Objects.requireNonNull(printfMethodHandle);
    var cString = memorySession.allocateUtf8String(str + "\n");
    return (int) printfMethodHandle.invoke(cString);
}

public static void main(String[] args) throws Throwable {
    var str = "Hello World";
    try (var memorySession = MemorySession.openConfined()) {
        System.out.println(printf(str, memorySession));
    }
}
```

### 5. Putting things together!

What we've learned by this point is that a memory session (**MemorySession**) or a segment allocator (**SegmentAllocator**) are key APIs to perform memory allocation.
A memory session should be declared with a try-with-resources to achieve implicit memory de-allocation.
There are several options to allocate memory segments - through a segment allocator or a memory session directly.
The Linker, symbol lookup objects, value/memory layouts, and method handles are static objects.


## Summary

The goal of this article is two folds, to give an overview of the Foreign Function & Memory API by looking at how to invoke a simple C function from Java.
Achieving such a trivial example might seem a bit complicated but seeing the different steps is key to understanding how the Foreign Function & Memory API works.
The good news is that developers can rely on the [jextract](https://github.com/openjdk/jextract) tool to handle most of the Foreign Function & Memory machinery.
Jextract will be covered in an upcoming article.

Several points need to be tackled when invoking native code from Java using the Foreign Function & Memory API:
* Get the native library and its corresponding header files;
* Build a function descriptor in Java (**FunctionDescriptor**);
* Lookup native memory address of a function symbol, and create a method handle for it;
* Create a related method handle and confirm that it has been properly created (ex. if the native library is not in the system path,
  the lookup will fail and the return a method handle will be null);
* Decide how the application will allocate memory segments: through a segment allocator or a memory session, make sure the memory allocation technique is consistent throughout the codebase of an application.

## Code listing

You may find sources for this chapter here [github.com/denismakogon/openjdk-project-samples](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-1).
<script src="https://emgithub.com/embed.js?target=https%3A%2F%2Fgithub.com%2Fdenismakogon%2Fopenjdk-project-samples%2Fblob%2Fmaster%2Fsrc%2Fmain%2Fjava%2Fcom%2Fopenjdk%2Fsamples%2Fpanama%2Fpart_1%2FPrintfSimplified.java&style=github&showBorder=off&showLineNumbers=off&showFileMeta=on&showCopy=on"></script>
