---
layout: post
title:  'Introduction to Project Panama. Part 2: Dynamic variadic functions.'
#date:   2022-06-21
categories: openjdk panama
tags: ["openjdk", "panama"]
image_src_url: 'https://unsplash.com/photos/Wiwqd_8Rds8/download?ixid=MnwxMjA3fDB8MXxzZWFyY2h8OXx8cGFuYW1hfGVufDB8fHx8MTY1NDMyNzMwMg&force=true&w=800'
excerpt: 'This article explores problems of variadic functions and possible implementations of variadic functions using Foreign Function and Memory API.'
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Intro

[Part 2]() shown how implement the C variadic functions in Java using Foreign Function and Memory API: simply define a function descriptor with variadic argument layouts in advance prior the invocation:
```java
FunctionDescriptor descriptorWithNamedAndVariadicArg = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
).asVariadic(
        ADDRESS.withBitAlignment(64), JAVA_INT.withBitAlignment(32)
);
```

There's one problem: it's required to define a function descriptor (and a method handle, eventually) in advance for every possible combination of the variadic arguments.
The code base of an application that uses native calls will potentially.

It may seem like the only flexible enough solution is a method overloading:
```java
public static int printf(String formatter) throws Throwable {
    var allocator = SegmentAllocator.implicitAllocator();
    var localDescriptor = FunctionDescriptor.of(JAVA_INT, ADDRESS);
    var printfMethodHandle = symbolLookup.lookup("printf").map(
        addr -> linker.downcallHandle(addr, localDescriptor)
    ).orElse(null);
    return (int) printfMethodHandle.invoke(allocator.allocateUtf8String(formatter));
}

public static int printf(String formatter, String arg1, String arg2) throws Throwable {
    var allocator = SegmentAllocator.implicitAllocator();
    var localDescriptor = FunctionDescriptor.of(JAVA_INT, ADDRESS).asVariadic(ADDRESS, ADDRESS);
    var printfMethodHandle = symbolLookup.lookup("printf").map(
        addr -> linker.downcallHandle(addr, localDescriptor)
    ).orElse(null);
    return (int) printfMethodHandle.invoke(
        allocator.allocateUtf8String(formatter),
        allocator.allocateUtf8String(arg1),
        allocator.allocateUtf8String(arg2),
    );
}
```
but only for that type of native functions where a scope of variadic arguments is limited, i.e., for functions that aren't like the C _printf_.

This article will focus on the complex but flexible solution -- dynamic variadic argument layouts.

## Dynamic variadic arguments

A term "dynamic" implies to an at attempt to guess what kind of value layouts will match objects passed down as varargs to a Java method.
Such Java method must be compliant with the C signature of the same native function. In the case of C _printf_:
```java
// int printf(const char * __restricted, ...);
int printf(Addressable x0, Object... x1);
```

An algorithm that implement dynamic variadic arguments is based on the fact that named argument layouts doesn't change over the time (unless a new version of a library introduced API changes).
So, it's possible to set in stone a part of the function descriptor that is responsible for a return value type and named arg layouts:
```java
// static base descriptor, will not change unless C _printf_ will
var base = FunctionDescriptor.of(JAVA_INT, ADDRESS);
```
while the only thing that impact a descriptor is the content of varargs (`Object... x1`).


The problem is that the JVM has no idea what the exact signature of a C function is, it only relies on the expected method handle type derived from a function descriptor.
Interesting fact, a descriptor holds more than just a return value and argument layouts, but also can tell how many arguments (both named and variadic) it has and a position of a first variadic argument.
It also means that the runtime has enough knowledge to which part of a method type corresponds to named or variadic arguments (based on the value of [**FunctionDescriptor::firstVariadicArgumentIndex**](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html#firstVariadicArgumentIndex())):
```java
var desriptor = FunctionDescriptor.of(JAVA_INT, ADDRESS).asVariadic(ADDRESS, int, long);
var indexOffirstVariadicArgument = desriptor.firstVariadicArgumentIndex(); // 1
```
```shell
     0              1       2    3
(Addressable, Addressable, int, long)int
 --------->  |       varargs       |
```

The general idea is to create such method handle proxy which method type correspond to the base part of a function descriptor with the additional **Object[]** vararg holder, but acting as an argument array-collecting method:
```java
methodHandle.invoke(namedArg1, ..., namedArgN, varArg1, ..., varArgN) ---> methodHandle.invoke(Object[] args)
```
i.e., it's necessary create a method handle that in a runtime can can collect whatever arguments it invoked with into an array of **Object**'s which method type correspond to a function descriptor with the additional named argument **Object[]**.

### Invoke method handle

Taking into account that a proxy suppose to be a generic solution for **ALL** variadic functions, it must learn from a function descriptor it creates a method handle for.
So, a proxy depends on how many argument a descriptor hold, their order and the type layouts, and subsequently the corresponding method type.
These aspects help to create a proxy method handle of a specific method type assuming that the last argument would be an array **Object[]** holding the variadic arguments:
```java

static {
    try {
        INVOKE_MH = MethodHandles.lookup().findVirtual(VarargsInvoker.class, "invoke", 
            MethodType.methodType(Object.class, SegmentAllocator.class, Object[].class));
    } catch (ReflectiveOperationException e) {
        throw new RuntimeException(e);
    }
}

static MethodHandle make(Linker linker, MemorySegment symbol, FunctionDescriptor function) {
    VarargsInvoker invoker = new VarargsInvoker(linker, symbol, function);
    MethodHandle handle = INVOKE_MH
                .bindTo(invoker)
                .asCollector(Object[].class, function.argumentLayouts().size() + 1);
    MethodType mtype = MethodType.methodType(
                function.returnLayout().isPresent() ?
                carrier(function.returnLayout().get(), true) :
                void.class
    );
    for (MemoryLayout layout : function.argumentLayouts()) {
        mtype = mtype.appendParameterTypes(carrier(layout, false));
    }
    mtype = mtype.appendParameterTypes(Object[].class);
    if (mtype.returnType().equals(MemorySegment.class)) {
        mtype = mtype.insertParameterTypes(0, SegmentAllocator.class);
    } else {
        handle = MethodHandles.insertArguments(handle, 0, THROWING_ALLOCATOR);
    }
    return handle.asType(mtype);
}
```

To make a proxy method handle versatile for all variadic functions it must be agnostic to a combination of both named and variadic arguments it might be invoked with, that's why it must act as an [argument array-collecting method](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodHandle.html#asCollector(java.lang.Class,int)) - 
simpli enclose all parameters into an array of type **Object[]**.

Note: It's definitely possible that a native function return a value type may not be a primitive type (like C _printf_), so to access a return value it's not needed to allocate a memory segment.
However, if a function returns a struct or an array it's necessary to allocate a memory segment to make this value accessible from the Java runtime, that's why the first argument in invocation prior to **Object[]** could be a **SegmentAllocator**.

The code above creates a method handle for the **VarargsInvoker::invoke** that act as a proxy between a consumer of a native function and the actual invocation.
It means that previously a method handle used to invoke a function was provided by a **Linker**:
```shell
MethodHandle printfHandle = LINKER.downcallHandle(symbol, descriptor); // a method handle of the C printf
```
But this time a consumer of a native function receive a method handle to a "proxy" method responsible for the downcall:
```java
MethodHandle proxyHandle = VarargsInvoker.make(symbol, descriptor); // a method handle of the VarargsInvoker::invoke
```

### Invoke method

Collecting all invocation parameters into the **Object[]** array eases further processing of parameters making the assumption that an array holds named parameters followed by an **Object[]** array holding variadic ones.
So, the **VarargsInvoker::invoke** is responsible for a couple of things:
1. Making sure a number of named parameters match to a number of layouts stored in the base function descriptor. That's the reason why a proxy keeps a copy of the base function descriptor:
```java
    int nNamedArgs = function.argumentLayouts().size();
    assert(args.length == nNamedArgs + 1);
    // The last argument is the array of vararg collector
    Object[] unnamedArgs = (Object[]) args[args.length - 1];

    int argsCount = nNamedArgs + unnamedArgs.length;
    Class<?>[] argTypes = new Class<?>[argsCount];
    MemoryLayout[] argLayouts = new MemoryLayout[nNamedArgs + unnamedArgs.length];
```
2. Creating a new function descriptor from both named and variadic parameters:
```java
    for (pos = 0; pos < nNamedArgs; pos++) {
        argLayouts[pos] = function.argumentLayouts().get(pos);
    }
    assert pos == nNamedArgs;
    for (Object o: unnamedArgs) {
        argLayouts[pos] = variadicLayout(normalize(o.getClass()));
        pos++;
    }
    assert pos == argsCount;
    FunctionDescriptor f = (function.returnLayout().isEmpty()) ?
        FunctionDescriptor.ofVoid(argLayouts) :
        FunctionDescriptor.of(function.returnLayout().get(), argLayouts);
```
The previous article shown a use of the **FunctionDescriptor::asVariadic** for variadic arguments as a part of the simplified implementation. 
However, it's not necessary to apply it here because there's no impact a final method type.

3. Invoking a native function with an array-collected parameters:
```java
return mh.asSpreader(Object[].class, argsCount).invoke(allArgs);
```

Full code listing can be found [here]().

### C _printf_ variadic

So, the implementation described above allow to implement a full contract of the C variadic functions.
However, the **MethodHandle** API it uses isn't trivial. At the same time an introduction of a method handle proxy 
between a consumer of a native function the actual invocation have opened a chance to manipulate by the aspects of a method handle, i.e., 
change a method type and turn a handle into an array-collecting.

The best way to showcase a power of such implementation is revising the C _printf_ implementation:
```java
private static final FunctionDescriptor printfDescriptor = FunctionDescriptor.of(
    JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);

private static void printf(MemorySegment namedArg, Object... varargs) {
    symbolLookup.lookup("printf").ifPresent(addr -> {
        MethodHandle printfHandle = VarargsInvoker.make(linker, addr, printfDescriptor);
        try {
            System.out.println(
                    (int) printfHandle.invoke(namedArg, varargs)
            );
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    });
}
```

So, 

## Summary

Part 1 covered a simplified version of C _printf_ that omitted a couple of critical moments,
one of them is variadic functions that are a normal practice in a software development daily routine.

It may look quite complex when you see it for the first time,
but it gets much better when you understand what are the critical issues and
how to build a unified interface between Java code and native variadic functions.

The good thing about **VarargsInvoker** implementation is it can be used in all possible scenarios:
a function with variadic only, named and variadic and, named only or no args.

I know that such solutions go beyond ordinary software development,
but things will get much and much simpler when you will introduce a new code tool to your portfolio - jextract!

That's what I'm going to talk about in the next Part 3!

## Code listings

### Appendix 1: Printf variadic

### Appendix 2: VarargsInvoker

Disclaimer: This code is a modified version of **VarargsInvoker** that is a part generated sources.
Technically, they belong to Panama but are not a part of the JDK or code tools.
Generated sources are not licensed.

```java
package com.openjdk.samples.panama.stdlib;

import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;

import static java.lang.foreign.ValueLayout.*;


class VarargsInvoker {
    private static final MethodHandle INVOKE_MH;
    private final MemorySegment symbol;
    private final FunctionDescriptor function;

    private VarargsInvoker(MemorySegment symbol, FunctionDescriptor function) {
        this.symbol = symbol;
        this.function = function;
    }

    static {
        try {
            INVOKE_MH = MethodHandles.lookup().findVirtual(VarargsInvoker.class, "invoke", 
                        MethodType.methodType(Object.class, SegmentAllocator.class, Object[].class));
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException(e);
        }
    }

    static MethodHandle make(MemorySegment symbol, FunctionDescriptor function) {
        VarargsInvoker invoker = new VarargsInvoker(symbol, function);
        MethodHandle handle = INVOKE_MH.bindTo(invoker).asCollector(Object[].class, function.argumentLayouts().size() + 1);
        MethodType mtype = MethodType.methodType(function.returnLayout().isPresent() ? carrier(function.returnLayout().get(), true) : void.class);
        for (MemoryLayout layout : function.argumentLayouts()) {
            mtype = mtype.appendParameterTypes(carrier(layout, false));
        }
        mtype = mtype.appendParameterTypes(Object[].class);
        if (mtype.returnType().equals(MemorySegment.class)) {
            mtype = mtype.insertParameterTypes(0, SegmentAllocator.class);
        } else {
            handle = MethodHandles.insertArguments(handle, 0, THROWING_ALLOCATOR);
        }
        return handle.asType(mtype);
    }

    static Class<?> carrier(MemoryLayout layout, boolean ret) {
        if (layout instanceof ValueLayout valueLayout) {
            return (ret || valueLayout.carrier() != MemoryAddress.class) ?
                    valueLayout.carrier() : Addressable.class;
        } else if (layout instanceof GroupLayout) {
            return MemorySegment.class;
        } else {
            throw new AssertionError("Cannot get here!");
        }
    }

    private Object invoke(SegmentAllocator allocator, Object[] args) throws Throwable {
        // one trailing Object[]
        int nNamedArgs = function.argumentLayouts().size();
        assert(args.length == nNamedArgs + 1);
        // The last argument is the array of vararg collector
        Object[] unnamedArgs = (Object[]) args[args.length - 1];

        int argsCount = nNamedArgs + unnamedArgs.length;
        Class<?>[] argTypes = new Class<?>[argsCount];
        MemoryLayout[] argLayouts = new MemoryLayout[nNamedArgs + unnamedArgs.length];

        int pos = 0;
        for (pos = 0; pos < nNamedArgs; pos++) {
            argLayouts[pos] = function.argumentLayouts().get(pos);
        }

        assert pos == nNamedArgs;
        for (Object o: unnamedArgs) {
            argLayouts[pos] = variadicLayout(normalize(o.getClass()));
            pos++;
        }
        assert pos == argsCount;

        FunctionDescriptor f = (function.returnLayout().isEmpty()) ?
                FunctionDescriptor.ofVoid(argLayouts) :
                FunctionDescriptor.of(function.returnLayout().get(), argLayouts);
        MethodHandle mh = LINKER.downcallHandle(symbol, f);
        if (mh.type().returnType() == MemorySegment.class) {
            mh = mh.bindTo(allocator);
        }
        // flatten argument list so that it can be passed to an asSpreader MH
        Object[] allArgs = new Object[nNamedArgs + unnamedArgs.length];
        System.arraycopy(args, 0, allArgs, 0, nNamedArgs);
        System.arraycopy(unnamedArgs, 0, allArgs, nNamedArgs, unnamedArgs.length);

        return mh.asSpreader(Object[].class, argsCount).invoke(allArgs);
    }

    private static Class<?> unboxIfNeeded(Class<?> clazz) {
        if (clazz == Boolean.class) {
            return boolean.class;
        } else if (clazz == Void.class) {
            return void.class;
        } else if (clazz == Byte.class) {
            return byte.class;
        } else if (clazz == Character.class) {
            return char.class;
        } else if (clazz == Short.class) {
            return short.class;
        } else if (clazz == Integer.class) {
            return int.class;
        } else if (clazz == Long.class) {
            return long.class;
        } else if (clazz == Float.class) {
            return float.class;
        } else if (clazz == Double.class) {
            return double.class;
        } else {
            return clazz;
        }
    }

    private Class<?> promote(Class<?> c) {
        if (c == byte.class || c == char.class || c == short.class || c == int.class) {
            return long.class;
        } else if (c == float.class) {
            return double.class;
        } else {
            return c;
        }
    }

    private Class<?> normalize(Class<?> c) {
        c = unboxIfNeeded(c);
        if (c.isPrimitive()) {
            return promote(c);
        }
        if (MemoryAddress.class.isAssignableFrom(c)) {
            return MemoryAddress.class;
        }
        if (MemorySegment.class.isAssignableFrom(c)) {
            return MemorySegment.class;
        }
        throw new IllegalArgumentException("Invalid type for ABI: " + c.getTypeName());
    }

    private MemoryLayout variadicLayout(Class<?> c) {
        if (c == long.class) {
            return JAVA_LONG;
        } else if (c == double.class) {
            return JAVA_DOUBLE;
        } else if (MemoryAddress.class.isAssignableFrom(c)) {
            return ADDRESS;
        } else {
            throw new IllegalArgumentException("Unhandled variadic argument class: " + c);
        }
    }
}
```
