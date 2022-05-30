---
layout: post
title:  'Introduction to OpenJDK Project Panama. Part 2. Variadic functions.'
date:   2022-06-14
categories: openjdk panama
tags: ["openjdk", "panama"]
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

[Introduction to OpenJDK Project Panama. Part 1.]({{ '/openjdk/panama/2022/05/29/introduction-to-project-panama-part-1.html' | relative_url }})

## "Java-varargs to C-variadic" problem

### C _printf_ is like _System.out::printf_

In [Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }}), you’ve learned how to performa a downcall to native code using Foreign Function and Memory Access APIs.
Indeed, learning curve is pretty long, however, it’s not like the biggest issue, once you’ll go through it - it would be clear to you, that it was not the biggest problem comparing to what JNI was to most developers working with native code.


### Revisiting _printf_ example

In [Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }}), I’ve shown you a simplified version of _printf_ that could be actually bound to Java’s _System.out::println_ 
rather than to _System.out::printf_. What I did is I moved varargs out of scope of a hello-world application. 
The reason for such move was a level of complex technical solutions that we need to dive into 
(potentially, the complexity goes beyond typical scope of ordinary routine Java application development).

So, how can we turn Java varargs into C varargs? The answer is - through a reflection! 
Now you see why I didn’t include this part into a hello-world app, because it would become a no longer hello-world, 
but rather advanced Java development.

At this moment we need to go back and look at couple things, first thing is the _printf_ **FunctionDescriptor**, 
it has a following look:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
There is no sign of varargs, right? This was done on purpose because at the moment of a **FunctionDescriptor** 
declaration we don’t know exactly what kind of varargs we’ll get at an invocation stage.

Important note: the downcall works only with instances of classes that implement **Addressable** interface: 
**MemorySegment** or **MemoryAddress**, these two Java types directly maps to a native memory segment (primitive types, structs) 
or an address of it (pointers). So, before submitting these varargs we also need to make sure they aren’t 
malformed and could be processed further down the pipe.

So, we perform perform a downcall to _print_ exactly as we declare it in **FunctionDescriptor** - 
_int_ as return type, _char *_ as an argument:
```java
return (int) printfMethodHandle.invoke(cString);
```

Assuming that in C we have variadic functions, we need to make sure that **MethodHandle::invokeExact** can accept varargs:
```java
public final native @PolymorphicSignature Object invokeExact(Object... args) throws Throwable;
```

As can you see, it doesn’t make a distinction between ordinary function args and varargs - 
it only accepts the varargs, but that’s exactly what we need. However, this method performs a downcall that **exactly** 
matches to a function descriptor. It means you need to have a function descriptor for any type of a varargs you’re about to pass.

From a software development standpoint, such approach eliminates the flexibility of variadic functions. 
So, our job is to preserve the same developer experience as they had with C, 
but this time writing a code full of downcalls in Java. That’s what we’re going to do through this post!

## "Java-varargs to C-variadic" solution

### Problematic area

#### **FunctionDescriptor** is not flexible

As I outlined above, the biggest issue is that we can't predict a set of varargs that'll be used in a downcall, therefore,
we can't create a generic version of a **FunctionDescriptor** that will _just work_ for every possible invocation configuration. 
So, if we can't build one then maybe we try creating a new function descriptor that extends the original descriptor for each invocation, dynamically.

#### **MethodHandle** depends on a **FunctionDescriptor**

A subsequent issue is that a native function method handle is provisioned from two objects: a native memory address and a function descriptor.
As of now, we already know that we need a new descriptor version for every combination of named args and varargs.

Look at _printf_ method handle definition:
```java
MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
        addr -> linker.downcallHandle(addr, printfDescriptor)
    ).orElse(null);
```
A method handle depends on a descriptor!Taking into account that both method handle and a descriptor should 
potentially be considered as static objects, we're eventually found ourselves in a situation when both of them aren't static any longer.

### Advanced **MethodHandle** usage

Before jumping into the pool with alligators I'd like to show you a very simple application that implements the reflective access
to a static member of a class.

#### Reflective access to members of a class

Imagine a hello-world application, nothing special, a static function, an invocation of it from _main_, like a typical straightforward way of calling static member of a class.
But there's one more way of call _helloWorld_ method without directly invoking it, but through its **MethodHandle**, 
i.e., accessing to a member of a class using a method handle lookup, also known as a reflective member access operation,
here's what I'm talking about:
```java
package com.openjdk.samples.panama.stdlib;

import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;

class MethodTypeExample {

    private static final MethodHandle helloWorld_mh;

    private static void helloWorld() {
        System.out.println("hello-world");
    }

    static {
        try {
            helloWorld_mh = MethodHandles.lookup().findStatic(
                    MethodTypeExample.class, "helloWorld",
                    MethodType.methodType(void.class));
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) throws Throwable {
        helloWorld_mh.invokeExact();
    }

}
```

The purpose of this code above is to find a method handle of a static member _helloWorld_ within a **MethodTypeExample.class** 
which method type matches to a _void function with no argument_. If a lookup was successful, a method _helloWorld_ can 
be called through its method handle.


### Creating reflected version on `invoke` method

We know that **MethodHandle::invokeExact** only accepts varargs, even though it's not really an issue, 
but it's way better to have a specific version of `invoke` that matches a signature of a native function,
i.e., makes a clear distinction between named and unnamed arguments.

So, the idea is to create an `invoke` method of the following signature:
```java
Object invoke(SegmentAllocator allocator, Object[] args);
```
that we can update in a runtime with a corresponding function descriptor plus varargs (of type **Object[].class**).

At the same time `invoke` method must be responsible for creating a new function descriptor, 
flattening both named and unnamed arguments into a form of **Object[]** to pass it down to a **MethodHandle::invokeExact**.

#### Reflecting redefinition of `invoke` based on a function descriptor

The reflecting redefinition algorythm suppose to look like:

* through a reflection, create a method handle for `VarargsInvoker::invoke` method with the corresponding method type:
```java
    static {
        try {
            INVOKE_MH = MethodHandles.lookup().findVirtual(
                VarargsInvoker.class, "invoke",
                MethodType.methodType(Object.class, SegmentAllocator.class, Object[].class)
            );
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException(e);
        }
    }
```

* bind an **VarargsInvoker::invoke** method handle to an instance of **VarargsInvoker** class:
```java
    MethodHandle handle = INVOKE_MH
            .bindTo(invoker)
            .asCollector(Object[].class, function.argumentLayouts().size() + 1);
```

* create a method type that corresponds to a function descriptor in a format of an `VarargsInvoker::invoke` signature, starting from a return value layout:
```java
    MethodType mtype = MethodType.methodType(
        function.returnLayout().isPresent() ? carrier(function.returnLayout().get(), true) : void.class
    );
```
in a context of _printf_, the ternary operator will give us an `int.class` while `mtype` will be hold an _int_ type representation as a return argument.


* append all class types based on the named args layouts to a method type:
```java
    for (MemoryLayout layout : function.argumentLayouts()) {
        mtype = mtype.appendParameterTypes(carrier(layout, false));
    }
```

* don't forget to add one more argument - varargs:
```java
mtype = mtype.appendParameterTypes(Object[].class);
```

* with the respect to **VarargsInvoker::invoke** signature, add a segment allocator as a first argument:
```java
mtype = mtype.insertParameterTypes(0, SegmentAllocator.class);
```

* set a native function descriptor-based method type to a method handle:
```java
return handle.asType(mtype); 
```

By this moment we've built an `invoke` method that maps one-to-one to function descriptor, but with additional **Object[].class** as varargs type.
In the context of _printf_, vararg types for `invoke` would have the following look:
```java
new Object[] {int.class, MemorySegment.class, Object[].class};
```

Note: **VarargsInvoker** implementation available in Appendix 2.


#### `invoke` routine

As I mentioned above, `invoke` must be responsible for two things:
- recreate a function descriptor based on the original function descriptor and the actual varargs;
- flattening both named args and varargs to pass them down to **MethodHandle::invoke** in a format of **Object[]**.

`invoke` algorythm looks like this:
* create a **MemoryLayout** array which size suppose to correspond to a number of both named args and varargs:
```java
    int nNamedArgs = function.argumentLayouts().size();
    assert(args.length == nNamedArgs + 1);
    Object[] unnamedArgs = (Object[]) args[args.length - 1];
    int argsCount = nNamedArgs + unnamedArgs.length;
    MemoryLayout[] argLayouts = new MemoryLayout[nNamedArgs + unnamedArgs.length];
```
* go through named args and varargs, fill a **MemoryLayout** array with the actual memory layouts of each arg:
```java
    int pos;
    for (pos = 0; pos < nNamedArgs; pos++) {
        argLayouts[pos] = function.argumentLayouts().get(pos);
    }

    for (Object o: unnamedArgs) {
        argLayouts[pos] = variadicLayout(normalize(o.getClass()));
    }
```
* create a new function descriptor and a method handle:
```java
    FunctionDescriptor f = (function.returnLayout().isEmpty()) ?
    FunctionDescriptor.ofVoid(argLayouts) :
    FunctionDescriptor.of(function.returnLayout().get(), argLayouts);
    MethodHandle mh = LINKER.downcallHandle(symbol, f);
```

* create a flat array containing both named args and varargs:
```java
    Object[] allArgs = new Object[nNamedArgs + unnamedArgs.length];
    System.arraycopy(args, 0, allArgs, 0, nNamedArgs);
    System.arraycopy(unnamedArgs, 0, allArgs, nNamedArgs, unnamedArgs.length);
```

* make a method handle accept **Object[]** of a fixed size, then perform an invocation of a native function with flattened args.
```java
    return mh.asSpreader(Object[].class, argsCount).invoke(allArgs);
```

The whole idea is to hijack an invocation to a native method through its method handle in order build a derived 
version of the original function description based on the given varargs. 

Note: **VarargsInvoker** implementation available in Appendix 2.


## Using **VarargsInvoker**

Finally, we are at THAT moment when we can finally can use the **VarargsInvoker**!
I'll skip a part where we create a linker, symbol lookups and so on (they remain the same from Part 1), 
but still there are two topics to cover:
* memory segments and addresses allocation;
* invocation;

### Invocation

We still need to look up a native address of a function, but instead of creating a **MethodHandle** using an instance of a **Linker**,
I will use **VarargsInvoker::make** method:
```java
    private static void printf(MemorySegment namedArg, Object... varargs) {
        symbolLookup.lookup("printf").ifPresent(addr -> {
            MethodHandle printfHandle = VarargsInvoker.make(addr, printfDescriptor);
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
A few things have changed, of course, you see varargs of a type **MethodAddress**. 
If you'll look at **Varargs::variadicLayout** you'll notice that there are only three type can be processed further: _JAVA_LONG_, _JAVA_DOUBLE_ and **MemoryAddress**.
The reason for such "limitation" is that the rest of Java types like **String**, **Integer** and so on, require a native memory allocation. 
Therefore, for _printf_, I have limited a scope of types to **MemoryAddress** only as the code operates by **String** variables that require native memory segment allocation.

### Memory segment and address allocations

What I'm trying to showcase, in C, would look like:
<script style="width: 1px;max-width: 100%;min-width: 100%;overflow: hidden;" src="//onlinegdb.com/embed/js/CfJgjmy8-?theme=dark"></script>

So, the _main_ would have the following representation:
```java
    public static void main(String[] args) throws Throwable {
        var parts = List.of("hello", "world", "from the", "other side");
        var stringFormat = "%s %s, %s %s!\n";
        var stringWithDecimals = "Hello, my name is %s, I'm %d years old.\n";
        try (var memorySession = MemorySession.openConfined()) {
            var varargs = parts.stream().map(
                p -> memorySession.allocateUtf8String(p).address()
            ).toList().toArray();
    
            printf(memorySession.allocateUtf8String(stringFormat), varargs);
    
            var newParts = new Object[]{
                memorySession.allocateUtf8String("Denis").address(),
                (long) 31
            };
            printf(memorySession.allocateUtf8String(stringWithDecimals), newParts);
        }
    }

```
The code above is pretty clear, it creates Java **String** variables.
Allocates a native memory for each of them, but for varargs - it grabs addresses to comply with a Java version of _printf_ function.

The output of a program is:
```shell
hello world, from the other side!
34
Hello, my name is Denis, I'm 31 years old.
43
```

## Summary

In Part 1, I covered a simplified version of C _printf_ that omitted a couple of critical moments, 
one of them is variadic functions that are a normal practice in a software development daily routine.

It may look quite complex when you see it for the first time, 
but it gets much better when you understand what are the critical issues and 
how to build a unified interface between Java code and native variadic functions.

The good thing about **VarargsInvoker** implementation is it can be used in both scenarios - when working with both 
variadic and named-args-only native functions.

## Code listings

### Appendix 1: Printf variadic

```java
package com.openjdk.samples.panama.stdlib;

import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;
import java.util.List;

import static java.lang.foreign.ValueLayout.ADDRESS;
import static java.lang.foreign.ValueLayout.JAVA_INT;


public class PrintfVariadic {
    private static final Linker linker = Linker.nativeLinker();
    private static final SymbolLookup linkerLookup = linker.defaultLookup();
    private static final SymbolLookup systemLookup = SymbolLookup.loaderLookup();
    private static final SymbolLookup symbolLookup = name ->
            systemLookup.lookup(name).or(() -> linkerLookup.lookup(name));
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

    public static void main(String[] args) throws Throwable {
        var parts = List.of("hello", "world", "from the", "other side");
        var stringFormat = "%s %s, %s %s!\n";
        var stringWithDecimals = "Hello, my name is %s, I'm %d years old.\n";
        try (var memorySession = MemorySession.openConfined()) {
            var varargs = parts.stream().map(
                    p -> memorySession.allocateUtf8String(p).address()
            ).toList().toArray();

            printf(memorySession.allocateUtf8String(stringFormat), varargs);

            var newParts = new Object[]{
                    memorySession.allocateUtf8String("Denis").address(),
                    (long) 31
            };
            printf(memorySession.allocateUtf8String(stringWithDecimals), newParts);
        }
    }

}
```

### Appendix 2: VarargsInvoker

Disclaimer: this code IS a part generated classes, it belongs to Panama, but is not a part of the JDK or code tools.

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
            // a method handle for this::invoke
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
