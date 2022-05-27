---
layout: post
title:  "Introduction to OpenJDK Project Panama. Part 1."
date:   2022-05-30
categories: panama openjdk
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Intro

As of now, we’re less than 4 month away from the JDK 19 release.
Time to talk about Project Panama, what’s going to be available for developers and how can developers use it even now.

## Project Panama

Project Panama is meant to be a bridge between two world: the JVM and native code written in other languages like C/C++.

So, Panama consists of 4 components:
  * Foreign Memory Access (FMA) and Functions: [JEP 424](https://openjdk.java.net/jeps/424)
  * Foreign Linker: [JEP 389](https://openjdk.java.net/jeps/389)
  * Vector API: [JEP 338](https://openjdk.java.net/jeps/338)

## Where to start?

First, you would need the latest [OpenJDK early access build](https://jdk.java.net/19/). It already has necessary features we need to start coding.

## What you need to know

When you actually bury you hand into these APIs you will need to operate with a couple major entities:
* [Memory session](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySession.html) - a compartment for allocated memory segments within this scope;
* [Segment allocator](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SegmentAllocator.html) - an API to allocate memory segments within a memory session;
* [Memory segment](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemorySegment.html) and [its address](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/MemoryAddress.html) - a set of API classes to work with native memory and pointer to it;
* [Linker](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Linker.html) and a [symbol lookup](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/SymbolLookup.html) - a set of API classes to perform down- and upcalls.
* [Value layouts](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/ValueLayout.html) and [descriptors](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html) - an APIs to model foreign types (structures, primitives, function descriptors).

## Part 1
As further you dig, the more you see that in order to start with Panama properly, it's so critical to a good introduction
just to make sure you haven't misses important concepts, techniques and methodologies.

So, In part 1, I will only cover **Linker**, (briefly) **symbol lookup** methods and will touch a little native memory management.
These three major components of Panama is basically are building block for more in-depth development of programs consisted from Java and native code.

## Linker

So, here I will try to cover **Foreign Linker API** and why it’s quite important change to the JDK that will ease a life for most developers who work close with C/C++ code from Java.

Let's start from the very beginning: what Foreign Linker is?
From technical standpoint, we’re talking about a “bridge” between two binary interfaces: the JVM and C/C++ native code, 
also known as [C ABI](https://en.wikipedia.org/wiki/Application_binary_interface).

In a real life, developers will work with C/C++ native code in a format of shared libraries and a header files.
Even now, when you're running you hello-world Java application, the JVM actually talks to C stdlib to execute one of many C functions (keep this in mind, we'll get back here).

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
Linker offers a set of API methods designed to perform both native calls.

While the **Linker** is like your cellphone - call whoever you want to, just dial in a proper phone number.
The symbol lookup methods are like your phonebook - just provide a proper descriptor who you want to call!

The Linker needs a method handle bound to a native memory segment to peform a downcall, the **SymbolLookup** is a way to find it using downcall method signature that consists of a return type, name and parameters description.

Let’s look at a very simple example - classic “hello world” but in C-style:
```cpp
int     printf(const char * __restrict, ...)
```

## HelloWold C-style in Java

Here I'll go through every step that you need to build a fully functional hello-world application with every step explained in details, in the format of tasks.

### Task: find a native memory segment of a native function.

We need to do couple things, first, to find a native segment of the _printf_ function:
```java
SymbolLookup linkerLookup = linker.defaultLookup();
SymbolLookup systemLookup = SymbolLookup.loaderLookup();

SymbolLookup symbolLookup = name ->
        systemLookup.lookup(name).or(() -> linkerLookup.lookup(name));

Optional<MemorySegment> printfMemorySegment = symbolLookup.lookup("printf");
```

Since [_printf_](https://www.cplusplus.com/reference/cstdio/printf/) is a part of C stdlib, you don't have to expect that a lookup will fail.
However, in a real life, it's possible that a lookup may fail because a symbol is missing due to misconfigured system paths.
But I'll cover it the following articles.

### Task: build a descriptor of a function you are calling.

Second thing to do is to define a descriptor that consists of a result type and accepted parameters.
If you're into C language, you know that functions like _printf_ are called as variadic functions, in Java we call it varargs, 
in case of _printf_, it accepts a char pointer and varargs (I all cover it later).

Note: from Java API standpoint it doesn't really matter what type hides behind that pointer as, in C, all pointers are the same 64-bit memory segment.

To simplify "Part 1", I'll define a simplified version of **FunctionDescriptor** for _printf_:
```java
FunctionDescriptor printfDescriptor = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
    );
```

A descriptor basically says: a function that returns _int32_ and accepts a 64-bit pointer as a parameter.
As you can see, it almost corresponds to its C definition from [stdio.h](https://www.cplusplus.com/reference/cstdio/printf/).

If you'll go and check a javadoc for **FunctionDescriptor** you'll notice that a method
```java
FunctionDescriptor of(MemoryLayout resLayout, MemoryLayout... argLayouts)
```

actually accepts a memory layout as both return value type and a function's arguments, 
in our simplified case - it's a pointer, ideally it has to be a pointer and varargs, 
but it also could be whatever your function needs. We'll dig into memory layout later.


### Task: build a method handle from a function's native memory segment.

Then we need to map a native address to the actual method handle that we can invoke:
```java
MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
            addr -> linker.downcallHandle(addr, printfDescriptor)
    ).orElse(null);
```
What happens here is we build _printf_ method handle out of a native memory segment of _printf_ and its simplified descriptor.
It is necessary to perform a downcall handle or to a _null_ if a lookup wasn't really successful.


### Task: native memory allocation.

Okay, at this stage you may think have pretty much everything we need to perform the actual downcall, but that's not all, 
we need to somehow bind a Java objects to a native memory segments to make sure _printf_ can access them.
Here’s the moment when we step into the area of **Foreign Memory Access API** (FMA)!


Way back in time when I did C programming (in high school, honestly), a memory allocation and freeing was kinda painful 
because no matter how good you are you’d, eventually you will forget to allocate or free a memory which will cause a 
program to leak it or blow up with a segmentation fault.

In Java, we rely on the GC to take care of memory cleaning up. 
The FMA API helps to allocate a memory off-heap which is a crucial part of any native interop story!

The FMA API allows to allocate and access memory segments, its addresses and shape up contiguous memory 
regions, located either on- or off- the Java heap. 
In concepts of Panama, all allocated memory segments must be bound to a specific memory session (**MemorySession**), 
but that’s not all, let’s leave this topic until next time.
< resource_scope >

Last, but not least important part of a program is a segment allocator derived from a memory session. By itself, a scope doesn’t have an API to allocate memory segments, but SegmentAllocator utility class designed to do so:
```java
try (var memorySession = MemorySession.openConfined()) {
    // memory allocation happening here
}
```

As I was told by Panama developers there are so many ways to do what you need, especially in terms of memory allocation.
One of my preferred memory allocation methods is to use a memory segment allocator, because it acts like an anchor for all further allocations that would go through it
while MemorySession gives more low level API access, but in a context of hello-world app I use segment allocator: 
```java
SegmentAllocator allocator = SegmentAllocator.newNativeArena(memorySession);
```

Finally, to call the _printf_ we need to allocate _const char *_ memory segment within a memory session using allocator and pass it to the C _printf_:
```java
MemorySegment cString = allocator.allocateUtf8String(str + "\n");
```

With an allocated memory segment, we now can invoke a function:
```java
private static int printf(String str, SegmentAllocator allocator) throws Throwable {
    Objects.requireNonNull(printfMethodHandle);
    var cString = allocator.allocateUtf8String(str + "\n");
    return (int) printfMethodHandle.invoke(cString);
}

public static void main(String[] args) throws Throwable {
    var str = "hello world";
    try (var memorySession = MemorySession.openConfined()) {
        var allocator = SegmentAllocator.newNativeArena(memorySession);
        System.out.println(printf(str, allocator));
    }
}
```

### Task: get things together!

Let's summarize few key aspects of C programs written in Java:
* it is necessary to outline that you need to start a memory session only when you actually about to allocate memory segments, 
so, an entrypoint to program should start with a session declaration;
* Use a session within try-with-resource block to make sure that a session is closed once you're done;
* It's up to use to decide which API to use to allocate memory segments - through an allocator or a session directly. Although, if you dig into the implementation you'll notice the **SegmentAllocator** has a couple of session safety checks.
* Linker, symbol lookup objets, value/memory layouts and method handles are static object.


Full listing:
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

    private static int printf(String str, SegmentAllocator allocator) throws Throwable {
        Objects.requireNonNull(printfMethodHandle);
        var cString = allocator.allocateUtf8String(str + "\n");
        return (int) printfMethodHandle.invoke(cString);
    }

    public static void main(String[] args) throws Throwable {
        var str = "hello world";
        try (var memorySession = MemorySession.openConfined()) {
            var allocator = SegmentAllocator.newNativeArena(memorySession);
            System.out.println(printf(str, allocator));
        }
    }
}
```

Just imagine how much work you need to do just to call a simple _printf_ function, especially comparing it to Java-style hello-world one-liner!
Well, you don't really have to write all of that, look at a tool called [jextract](https://github.com/openjdk/jextract). 
However, a programs like this are crucial for understanding all entities and concepts that stand behind of a new way of talking to native code.


## Summary

So, there are a couple of main questions to be answered during software development of programs performing downcalls through a Linker and FMA API:
* Decide which library you’re going to use, find a header file to it, look for functions you need;
* Build a function descriptor in Java (**FunctionDescriptor**);
* Lookup a native memory segments of a function symbol, create a method handle for it;
* Make sure a method handle exists (not null), it may appear that a library you're going to use is not in a system path (lookup will fail);
* Define how would allocate memory segments, make sure you follow the same coding technique through your program.
