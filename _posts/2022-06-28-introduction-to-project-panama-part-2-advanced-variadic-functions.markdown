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
The fact that a native functions like the C _printf_ does not declare any limitations against variadic arguments makes the whole implementation either quite challenging or incomplete.

So, there is a strong demand for a solution that will be flexible enough to fit all needs to be declared by variadic functions.
This article will focus on the complex but flexible solution -- dynamic variadic argument layouts.

## Dynamic variadic arguments

A term "dynamic" implies to an at attempt to figure out what kind of value layouts will match parameters passed down as variadic arguments to the downcall.

The overall implementation is based on the fact that named argument layouts doesn't change over the time (unless a new version of a library introduced API changes).
So, a part of the function descriptor responsible for a return value and named arg layouts is immutable.

The general idea is to create a wrapper and an array-collecting method handle for it which method type correspond to the base part of a function descriptor with the additional **Object[]** variadic arguments holder.

### Altering method handle

Every method and/or a function has its own signature consisting of named and variadic arguments, a number and the purpose of both may vary.
A function descriptor that represents a native function in the JVM is literally the only entity that hold knowledge regarding the amount, layouts and a position of a first variadic argument. 
At the same time, a method type created from a descriptor doesn't have a difference between variadic and named arguments - they all are member of type.

So, the idea is: no matter what was the combination of named and variadic arguments used to call a variadic native function it should be treated as ab **Object[]** array.
It means that a wrapper instance must alter its own method handle to become an [array-collecting](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodHandle.html#asCollector(java.lang.Class,int)) handle.
So, at invocation, a wrapper will gather both named and variadic arguments into an **Object[]** array, for instance:
```java
printfHandle.invoke(formatter, varArg1,varArg2, varArg3);
```
would turn into:
```java
printfHandle.invoke(new Object[] {formatter, varArg1,varArg2, varArg3});
```

### Altering method type

As stated above, an algorithm based on the fact that a part of descriptor responsible for a return value and named arg layouts doesn't change.
So, a wrapper can use the base part of a descriptor to create a method type:
```java
MethodType mtype = MethodType.methodType(
            function.returnLayout().isPresent() ?
            carrier(function.returnLayout().get(), true) :
            void.class
);
for (MemoryLayout layout : function.argumentLayouts()) {
    mtype = mtype.appendParameterTypes(carrier(layout, false));
}
```
However, such method type only correspond to named args. 
To fix that, the last argument in a method type must be defined as **Object[].class**, i.e., the simplest way to represent variadic arguments:
```java
mtype = mtype.appendParameterTypes(Object[].class);
```

Such definition of a method type imposes certain restrictions to the implementation of a native functions using Foreign Function and Memory API.
For instance, the definition if the C _printf_ would look like:
```java
public static int printf(Addressable x0, Object... x1);
```
Such restriction should not be considered as a limitation rather than an attempt to make these functions look exactly as they are defined in the C libraries.
In the case of C _printf_:
```cpp
int printf(const char * __restrict, ...);
```

### Creating a wrapper method handle

A downcall to a function consists of two stage:
* Creating a method handle.
* Invoking a native function.

So, a wrapper must follow the same logic. A wrapper init procedure must create an array-collecting method handle which method type derived from a function descriptor with the additional argument of type **Object[].class**:
```java
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
    MethodHandle handle = INVOKE_MH.bindTo(invoker)
        .asCollector(Object[].class, function.argumentLayouts().size() + 1);
    MethodType mtype = MethodType.methodType(function.returnLayout().isPresent() ? 
        carrier(function.returnLayout().get(), true) : void.class);
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

In the context of C _printf_, a wrapper method handle type would be:
```java
(Addressable, Object[].class)int
```

Note: It may be possible that a native function return a value type may not be a primitive type (like C _printf_), so to access a return value it's not needed to allocate a memory segment.
However, if a function returns a struct or an array it's necessary to allocate a memory segment to make this value accessible from the Java runtime, that's why the first argument in invocation prior to **Object[]** could be a **SegmentAllocator**.


### Invoking a method handle

Previous article shown that Java runtime will check if a method handle type defined in advance match to a method type created from parameters used to invoke a method handle.
It means that a call to a wrapper method handle require an array of type **Object[]** to be presented as the last argument even if there are no variadic arguments.
Simply wrapping a method handle invocation into a Java method that accept varargs makes such requirement less strict:
```java
private static void printf(Addressabke x0, Object... x1) {
    symbolLookup.lookup("printf").ifPresent(addr -> {
        MethodHandle printfHandle = VarargsInvoker.make(addr, printfDescriptor);
        try {
            return (int) printfHandle.invoke(x0, x1);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    });
}
```

Keeping in mind that a wrapper method handle is an array-collecting, all parameters used to invoke it will be gathered into an array.
So, the runtime responsibility of a wrapper to build the actual downcall method handle.

### Constructing downcall method handle

During the invocation, a wrapper responsible for creating a downcall method handle for a native function.
Part 1 explained that a method handle depends on a native symbol and a function descriptor.

So, to make a downcall successful a wrapper must:
* Unfold parameters array into one-dimension **MemoryLayout[]** array because variadic parameters added to an array in a form of another array:
```java
new Object[] {namedArg0, ..., namedArg, new Object[]{varArg1, ..., varArgN} } --> new MemoryLayout[] {namedArg0, ..., varArgN}
```

* Create a function descriptor. The **MemoryLayout[]** array basically hold positional argument layouts of a native function that a wrapper must call.
Therefore, a wrapper must construct a descriptor out if the layouts array:
```java
    FunctionDescriptor f = (function.returnLayout().isEmpty()) ?
        FunctionDescriptor.ofVoid(argLayouts) :
        FunctionDescriptor.of(function.returnLayout().get(), argLayouts);
```

Note: It's not necessary to use **FunctionDescriptor::asVariadic** here to define variadic arguments separately from 
named arg layouts because they eventually will become parts of a method type, one after another.

* Unfold invocation parameters into one-dimension array, similarly to the **MemoryLayout[]** array:
```java
new Object[] {namedArg0, ..., namedArg, new Object[]{varArg1, ..., varArgN} } --> new Object[] {namedArg0, ..., varArgN}
```

* Invoke a native function. To make an invocation successful it's required to spread arguments to match a method type created from a function descriptor (that was created from the invocation parameters):
```java
return mh.asSpreader(Object[].class, argsCount).invoke(allArgs);
```

This article described to how implement variadic argument for all variadic function, with no exceptions.
So, the idea of a dynamic variadic arguments is in re-working a function descriptor in the runtime. It indeed works, however, it has the downside.
A problem hides in-between parameters array flattening and a descriptor provisioning -- a wrapper attempts to guess variadic argument layouts:
```java
    for (Object o: unnamedArgs) {
        argLayouts[pos] = variadicLayout(normalize(o.getClass()));
        pos++;
    }
```

as well as changing an argument layout type based on a carrier class:
```java
    private Class<?> promote(Class<?> c) {
        if (c == byte.class || c == char.class || c == short.class || c == int.class) {
            return long.class;
        } else if (c == float.class) {
            return double.class;
        } else {
            return c;
        }
    }
```
i.e., _JAVA_LONG_ will correspond not only to _long_, but also for _byte_, _short_ and so on.


## Summary

It wouldn't be correct to make a conclusion only against this particular article. It would be much better to take into account Parts 1 & 2.

All these three articles grouped together by the idea of performing downcalls to C native code. 
Through Part 1 to up to this article a complexity of solutions and the amount of infrastructure code was growing pretty fast although the goal remained the same - to call the C _printf_.

Part 1 was almost the easiest one to explain and to implement, however, a solution was incomplete and barely useful.
Part 2 made a step forward towards a final goal, however, still had downsides (simplicity, code duplication) outnumbering the benefits.
This article almost hit the goal although the complexity skyrocketed that could be a deal-breaker but only at the cursory glance. 

So, the next article would be the cover a very important part of Project Panama - code generating.

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
