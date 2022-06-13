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

## Dynamic variadic arguments idea

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

The general idea is to create a dynamic array-collecting method handle proxy between
```java
invoke(Arg1, ..., ArgN, varArg1, ..., varArgN) ----> invoke(Objectp[] {Arg1, ..., ArgN, varArg1, ..., varArgN})
```

which method type correspond to a function descriptor. 
So, a call ot the variadic function suppose to look like:
```java
MethodHandle::invoke(namedArg1, ..., namedArgN, variadicArg1, ..., variadicArgN);
```

For instance, a call to the C _printf_ suppose to look like:
```java
VarargsInvoker(address, Function.of(JAVA_INT, ADDRESS)).invoke(formatter, variadicArg1, ..., variadicArgN);
```

### Arguments collector

So far the problem is that the JVM has no idea what the exact signature of a C function is, it only depends on a function descriptor.
Interesting fact, a function descriptor holds more than just a return value and argument layouts, but also can tell how many arguments (both named and variadic) it has and a position of a first variadic argument.

The problem is that the **MethodHandle::invoke** have no idea where named args end and variadic start.

Makes an array-collecting method handle, which accepts a given number of trailing positional arguments and collects them into an array argument. 
The new method handle adapts, as its target, the current method handle. The type of the adapter will be the same as the type of the target, 
except that a single trailing parameter (usually of type arrayType) is replaced by arrayLength parameters whose type is element type of arrayType.

So far the problem is that the JVM has no idea what the exact signature of a C function is, it only depends on a function descriptor.
Such dependency form can be expressed in two statements:


## Summary

In Part 1, I covered a simplified version of C _printf_ that omitted a couple of critical moments,
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

    private final Linker linker;
    private static final MethodHandle INVOKE_MH;
    private final MemorySegment symbol;
    private final FunctionDescriptor function;

    private static final SegmentAllocator THROWING_ALLOCATOR = (size, align) -> {
        throw new IllegalStateException("Cannot get here");
    };

    private VarargsInvoker(Linker linker, MemorySegment symbol, FunctionDescriptor function) {
        this.linker = linker;
        this.symbol = symbol;
        this.function = function;
    }

    static {
        try {
            INVOKE_MH = MethodHandles.lookup().findVirtual(VarargsInvoker.class, "invoke", MethodType.methodType(Object.class, SegmentAllocator.class, Object[].class));
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
        int nNamedArgs = function.argumentLayouts().size();
        assert(args.length == nNamedArgs + 1);
        Object[] unnamedArgs = (Object[]) args[args.length - 1];

        int argsCount = nNamedArgs + unnamedArgs.length;
        MemoryLayout[] argLayouts = new MemoryLayout[nNamedArgs + unnamedArgs.length];

        int pos;
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
        MethodHandle mh = linker.downcallHandle(symbol, f);
        if (mh.type().returnType() == MemorySegment.class) {
            mh = mh.bindTo(allocator);
        }
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
