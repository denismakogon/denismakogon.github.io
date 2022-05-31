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

## Part 1

Before you start working with Project Panama, it's critical to have a proper introduction, as many concepts aren't familiar to most Java developers. This will help make sure you under

As the further you dig, the more you see that to start working with Panama properly, it's so critical to have a good introduction
just to make sure you haven't missed important concepts, techniques, and methodologies.

So, in Part 1, I will only cover **Linker**, (briefly) **SymbolLookup** methods and will touch on a little native memory management.
These three major components of Panama are building blocks for more in-depth development of programs consisting of Java and native code.

## Linker

Let's start from the very beginning: what is Foreign Linker?
From a technical standpoint, we’re talking about a _“bridge”_ between two binary interfaces: the JVM and C/C++ native code, 
also known as [C ABI](https://en.wikipedia.org/wiki/Application_binary_interface).

In a real-life, developers will work with C/C++ native code in a format of shared libraries of a platform-specific format (`*.so, *.dylib, *.dll`) and their header files.
Even now, when you're running your hello-world Java application, the JVM talks to C stdlib to execute one of many C functions.

The JDK already offering a set of the C ABI implementation for all popular platforms (including latest macOS AARCH64):
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

In the JDK terminology, **Linker** is an instance of the platform-specific C ABI implementation.
Linker offers a set of API methods designed to perform both down- and upcalls.

While the **Linker** is like your cellphone - call whoever you want to, just dial in a proper phone number.
The symbol lookup methods are like your phonebook - just provide a proper descriptor of who you want to call!

To perform a downcall, you need a descriptor of a function you’d like to call, a native address allocated through a symbol lookup,
and a linker to create a method handle that you can invoke.

To make it clear, let's briefly look at what both down- and up-calls are:
* Downcall. [Wiki](https://en.wiktionary.org/wiki/downcall) says it's an event initiated from a high-level subsystem,
  in our case the JVM to a lower-level subsystem, like the OS kernel. I'll explain it later in the text in the context of Foreign Function & Memory API.
* Upcall. The opposite thing to a downcall.

Let’s implement a classic C-style “hello world” but in Java:
```cpp
int     printf(const char * __restrict, ...)
```

## HelloWold C-style in Java


### Task: find the address of the native function.

We need to do couple things, first, to look up for a native segment of the _printf_ function:
```java
SymbolLookup linkerLookup = linker.defaultLookup();
SymbolLookup systemLookup = SymbolLookup.loaderLookup();

SymbolLookup symbolLookup = name ->
        systemLookup.lookup(name).or(() -> linkerLookup.lookup(name));

Optional<MemorySegment> printfMemorySegment = symbolLookup.lookup("printf");
```

Since [_printf_](https://www.cplusplus.com/reference/cstdio/printf/) is a part of C stdlib, you don't have to expect that a lookup will fail.
However, in a real-life, a lookup may fail because a symbol is missing due to misconfigured system paths.
But I'll cover it in the following articles.

### Task: build a descriptor of a function you are calling.

Once we know where C _printf_ resides, the second thing to do is to define the _printf_ descriptor that consists of a result type and accepted parameters.
If you're into C language, you know that functions like _printf_ are called variadic functions, in Java we call it a function with varargs,
in the case of _printf_, it accepts a char pointer and varargs (I all cover it in the next part).

Note: from a Java API standpoint it doesn't matter what type hides behind that pointer in C all pointers have the same layout (which could be 32/64 but is fixed by the platform).

To simplify Part 1, I'll define a simplified version of **FunctionDescriptor** for _printf_:
```java
FunctionDescriptor printfDescriptor = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
    );
```

A descriptor says a function that returns _int32_ and accepts a 32/64-bit (fixed by a platform) pointer as a parameter.
As you can see, it _almost_ corresponds to its C definition from [stdio.h](https://www.cplusplus.com/reference/cstdio/printf/).

If you'll go and check a Javadoc for **FunctionDescriptor** you'll notice the following signature:
```java
FunctionDescriptor of(MemoryLayout resLayout, MemoryLayout... argLayouts)
```

a method **FunctionDescriptor::of** accepts a memory layout as both return value type and a function's arguments,
in our simplified case - it's a pointer, ideally, it has to be a pointer and varargs,
but it also could be whatever your function needs.

#### C to Java types through ValueLayout

You may ask: "wait, what `JAVA_INT` is?!". You're right, let's make a side-step and look at it thoroughly!
As per Javadoc, `JAVA_INT`
```java
// ValueLayout.class
OfInt JAVA_INT = new OfInt(ByteOrder.nativeOrder()).withBitAlignment(32);
```
is an instance of a value layout whose carrier is **int.class**.
With this layout we're telling to a **Linker** that we're creating a bridge between C _int32_
and Java instance of 32-bit _int_ with a carrier-class **int.class**. Later on, you'll see how it's all connected!

### Task: build a method handle from a function's native memory segment.

As of now, we alredy have a native address of _printf_, and its Java descriptor, so now we can create a method handle for C _printf_:
```java
MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
            addr -> linker.downcallHandle(addr, printfDescriptor)
    ).orElse(null);
```
What happens here is we build the _printf_ method handle out of a native memory segment of _printf_ and its simplified descriptor.
Upon the unsuccessful lookup, we potentially will get a null method handle which would be a signal of an issue.

So, at this stage, I have introduced all necessary terms to explain what both down- and upcalls are, in terms of Java -
_downcall_ is an invocation of a native function through a **MethodHandle** formed from a native function address and its Java version of a function descriptor.
At the same time, the _upcall_ is an invocation of some code written in Java through a **MethodHandle** converted
into a native memory segment which could then be passed to native functions as a function pointer.


### Task: allocate a native memory.

Okay, at this stage you may think we have pretty much everything we need to perform the actual downcall, but that's not all.
We need somehow to bind Java objects to native memory segments to make sure C _printf_ can access them.
Here’s the moment when we step into the area of **Foreign Memory Access API** (FMA)!

Way back in time when I did C programming (in high school, honestly), a memory allocation and freeing were kinda painful
because no matter how good you are you’d, eventually you will forget to allocate or free a memory which will cause a
program to leak it or blow up with a segmentation fault.

In Java, we traditionally relied on the GC to take care of memory cleaning up.
In Panama, the FMA API helps to allocate a memory off-heap which is a crucial part of any native interop story!

The FMA API allows to allocate and access memory segments, their addresses, and the shape of contiguous memory
regions, located either on- or off- the Java heap.
In concepts of Panama, all allocated memory segments must be bound to a specific memory session (**MemorySession**),
in other terms, consider a session like a unified memory allocation tool,
for instance, C `malloc`, but auto-closable (thanks to Java try-with-resources syntax block).

Last, but not least important part of a program is a segment allocator derived from a memory session.
By itself, a memory session serves the role of a native memory allocation API:
```java
try (var memorySession = MemorySession.openConfined()) {
    // memory allocation happening here
    SegmentAllocator allocator = SegmentAllocator.newNativeArena(memorySession);
}
```

As I was told by Panama developers, there are so many ways to do what you need in a right way especially in terms of memory allocation.
One of possible memory allocation methods in Java is **SegmentAllocator**, it is similar to **MemorySession**,
but often used in a context of other C memory allocation functions different from C [malloc](https://en.cppreference.com/w/c/memory/malloc).
For the simplicity of a hello-world application we will use **MemorySession** as a memory segment allocation tool.

Finally, to call C _printf_ we need to allocate _const char *_ memory segment within a memory session using **MemorySession** and pass it to C _printf_ function:
```java
MemorySegment cString = memorySession.allocateUtf8String(str + "\n");
```

With an allocated memory segment, we now can invoke a function:
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

Note that we are allowed to cast a return value to _int_, because of the value layout of a return value defined within _printf_ **FunctionDescriptor**.

In a real-life, instead of _int_ as a return value there possibly would be just a **MemorySegment** that you'd need to read
from to get a value of a type (a struct, for instance). I'll show you more examples in the following parts!

### Task: get things together!

Let's summarize a few key aspects of C programs written in Java:
* it is necessary to outline that you need to start a memory session only when you are actually about to allocate memory segments,
  so, an entry point to the program should start with a session declaration;
* Use a session within the try-with-resource block to make sure that a session is closed once you're done;
* It's up to use to decide which API to use to allocate memory segments - through an allocator or a session directly. Although, if you dig into the implementation you'll notice the **SegmentAllocator** has a couple of session safety checks.
* Linker, symbol lookup objects, value/memory layouts, and method handles are the static object.


The full code listing can be found below!

## Summary

Just imagine how much work you need to do just to call a simple _printf_ function, especially comparing it to a Java-style hello-world one-liner!
Well, you don't have to write all of that, look at a tool called [jextract](https://github.com/openjdk/jextract).
However, programs like this are crucial for understanding all entities and concepts that stand behind a new way of talking to native code.

So, there are a couple of main questions to be answered during the software development of programs performing downcalls through a **Linker** and FMA API:
* Decide which library you’re going to use, find a header file to it, and look for functions you need;
* Build a function descriptor in Java (**FunctionDescriptor**);
* Lookup native memory segments of a function symbol, and create a method handle for it;
* Make sure a method handle exists (not null), it may appear that a library you're going to use is not in a system path (lookup will fail);
* Define how would allocate memory segments, make sure you follow the same coding technique throughout your program.

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
