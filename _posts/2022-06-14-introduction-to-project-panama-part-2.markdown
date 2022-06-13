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

[Introduction to OpenJDK Project Panama. Part 1.]({{ '/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html' | relative_url }})

## "C variadic functions in Java" problem

### C _printf_ is Java _PrintStream::println_

[Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }}) 
covered very important downcall concept implemented within Project Panama, to be specific, as part of Foreign Memory and Foreign Function Interfae API.

As a practical task, in Part 1, I have shown you how to implement a downcall to C _printf_ function using corresponding API classes:
* **SymbolLookup** for searching native function address;
* **FunctionDescriptor** for declaring a function descriptor containing a return value and argument layouts;
* **Linker** for creating a method handle for a native function from a descriptor and a native function address;
* **MemorySession** for native memory segment allocations;

### Revisiting _printf_ example

[Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }}) 
contains simplified implementation of C _printf_ - it didn't have one key implementation aspect. The missing feature
turned _printf_ implementation into a _System.out::println_ rather than to _PrintStream::printf_.

Note: there is no _println_ in C standard library, because _println_ that we all know is the equivalent to _printf_ 
with the additional '\n' char in the end of a string printed to a console. 

In Java, the key difference _PrintStream::println_ and _PrintStream::printf_ is a presence of varargs in the last one. 
Such distinction was derived from C standard library. Now it's clear, the Part 1 did not implement a full contract of a variadic function C _printf_.

### Variadic functions, variadic arguments

In C/C++ world, variadic function is a function, like _printf_ which takes a variable number of argument indicated by the parameter of the form `...` 
which must appear last in the parameter list and must follow at least one named parameter:
```cpp
int     printf(const char * __restrict, ...);
```

These three dots are called variadic arguments. At the function call variadic args could be a list of args or nothing:
```cpp
printf("hello world");
printf(" I'm a=%s, I'm b=%s", a, b);
```

Note that it's okay to call variadic arguments as varargs, everyone will understand you, so as I will do the same through this article.

A decision to put varargs out of scope of the "Hello World" application was made due to the complexity of a necessary solution.

### Difference between normal and variadic functions

~~So, what's the process of invoking native variadic functions? The answer is - through a reflection!
Now you see why I did not include this part into a hello-world app because it would become a no longer hello-world,~~
~~but rather advanced Java development.~~


At this moment, we need to go back and look at a couple of things, first, is the _printf_ **FunctionDescriptor**:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
as you see, there is no sign of varargs. This was done on purpose because at the moment of declaring **FunctionDescriptor**
it's not possible know exactly what kind of vararg combination a function will get at an invocation stage.

An invocation of a native function is happening when a **MethodHandle::invoke** is called:
```java
return (int) printfMethodHandle.invoke(cString);
```

According to C _printf_ descriptor, it has both named args (`const char * __restricted`) and varargs (`...`), 
at the same time **MethodHandle::invoke** accepts varargs only:
```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
```
As can you see, it does not make a distinction between ordinary named args and varargs.

From a software development standpoint, our job is to preserve the same (or make it better) developer experience offered in C/C++,
but this time creating applications containing downcalls to native functions written in Java. 
That's what exactly this post will be about!

## "C variadic functions in Java" solution

### Predictable combination of varargs

It's necessary to revisit the definition of a function descriptro:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
as I mentioned before, such declaration only covers C _printf_ return value type and named args.

According to the **FunctionDescriptor** [JavaDoc](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html#asVariadic(java.lang.foreign.MemoryLayout...)),
it is possible to define the exact varargs for a function descriptor:
```java
public FunctionDescriptorPREVIEW asVariadic(MemoryLayoutPREVIEW... variadicLayouts)
```
it creates a specialized variadic function descriptor, by appending given variadic layouts to this function descriptor argument layouts.

Using this method it is possible to explicitly declare varargs memory layouts. 
Let's say we want C _print_ to accept _char * p_ and _int_ as varargs in addition to named args:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
        );
FunctionDescriptor variadicFunction = function.asVariadic(ADDRESS, JAVA_INT);
```

Such descriptor indeed compliant with the C _printf_ signature:
```cpp
int     printf(const char * __restrict, ...);
```
however, Java-based function descriptor holds more details than C _printf_ version.
It says that a function that corresponds to given descriptor returns _int_, accepts _char * p_ as named arg 
as well as **mandatory** varargs of a specific length (two, for example) and predefined types (C pointer and _int_).

java.lang.invoke.WrongMethodTypeException

### Unpredictable combinations of varargs

The biggest issue is that we can't predict a set of varargs that'll be used at invocation stage, therefore,
we can't create a generic version of a **FunctionDescriptor** that will _just work_ for every possible varargs configuration.
So, it's not possible build one then maybe it's manageable to create a new function descriptor based on the original descriptor for each invocation, dynamically.

### **FunctionDescriptor** is a key part of **MethodHandle**

A function descriptor is a key dependency of a native function method handle:
```java
MethodHandle printfMethodHandle = symbolLookup.lookup("printf").map(
        addr -> linker.downcallHandle(addr, printfDescriptor)
    ).orElse(null);
```

which means for every new function descriptor there has to be a new method handle.

### Advanced **MethodHandle** usage

To solve the varargs problem it is necessary to implement a method handle configured using the original function descriptor.
At the same time it must have a custom version of **MethodHandle::invoke** method that can dynamically 
build a new function descriptor from the original descriptor and given varargs at the execution stage.

A solution described above require to implement reflective access to a virtual member of a class. 
If you are familiar with the reflective access, then you may skip this part, if not - let's proceed.

#### Reflective access to members of a class

Imagine a hello-world application - _main_ and _helloWorld_ static methods, _main_ invokes _helloWorld_. 
Let's call _helloWorld_ through its method handle:
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

We know that **MethodHandle::invoke** accepts varargs, but there are no named args, even though it's not an issue,
but it's way better to have a specific version of `invoke` that matches a signature of a native function,
i.e., makes a clear distinction between named and unnamed arguments.

So, the idea is to dynamically create an `invoke` method that we can set up in runtime with a corresponding function
descriptor holding all necessary named arguments plus varargs (of type **Object[].class**).

At the same time `invoke` method must be responsible for creating a new function descriptor derived from the original one,
creating an array of type **Object[].class** containing named args followed by unnamed arguments, performing the actual downcall using **MethodHandle::invoke**.

#### `invoke` reflecting definition based on the original function descriptor

The idea is: in a runtime, create an `invoke` method that will comply with a native function C interface.
So, the algorithm supposes to look like this:

* through the reflective access to a virtual member of a class, create a method handle for the `VarargsInvoker::invoke`:
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

* create a method type that corresponds to a function descriptor starting from a carrier class standing behind the value layout:
```java
    MethodType mtype = MethodType.methodType(
        function.returnLayout().isPresent() ? carrier(function.returnLayout().get(), true) : void.class
    );
```
in the context of _printf_, the ternary operator will give us an `int.class` while `mtype` will behold an _int_ type representation as a return argument.

* append all class types based on the named args layouts to a method type:
```java
    for (MemoryLayout layout : function.argumentLayouts()) {
        mtype = mtype.appendParameterTypes(carrier(layout, false));
    }
```

* append varargs placeholder:
```java
mtype = mtype.appendParameterTypes(Object[].class);
```

* set a native function descriptor-based method type to a method handle:
```java
return handle.asType(mtype); 
```

By this moment we've built an `invoke` method that maps one-to-one to function descriptor, but with the additional **Object[].class** as varargs placeholder type.
In the context of _printf_, vararg types for `invoke` would have the following look:
```java
new Object[] {int.class, MemorySegment.class, Object[].class};
```

Note: complete **VarargsInvoker** implementation available in Appendix 2.


#### `invoke` responsibilities

As I mentioned above, `invoke` must be responsible for two things:
- recreating a function descriptor based on the original function descriptor and the actual varargs;
- creating an array of type **Object[].class** holding named args followed by varargs.

`invoke` routine looks like this:
* on the invocation, create a **MemoryLayout** array whose size suppose to correspond to a number of both named args and varargs:
```java
    int nNamedArgs = function.argumentLayouts().size();
    assert(args.length == nNamedArgs + 1);
    Object[] unnamedArgs = (Object[]) args[args.length - 1];
    int argsCount = nNamedArgs + unnamedArgs.length;
    MemoryLayout[] argLayouts = new MemoryLayout[nNamedArgs + unnamedArgs.length];
```
* loop through named args and varargs, fill a **MemoryLayout** array with the actual memory layouts of every arg:
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

* create a flat array containing both named args and varargs followed one by another:
```java
    Object[] allArgs = new Object[nNamedArgs + unnamedArgs.length];
    System.arraycopy(args, 0, allArgs, 0, nNamedArgs);
    System.arraycopy(unnamedArgs, 0, allArgs, nNamedArgs, unnamedArgs.length);
```

* configure a method handle to accept **Object[]** of a fixed size, then perform an invocation of a native function with an array of args.
```java
    return mh.asSpreader(Object[].class, argsCount).invoke(allArgs);
```

The whole idea is to hijack an invocation to a native method through its method handle in order to build a desired
the version of the original function description based on the given varargs.

Note: **VarargsInvoker** implementation available in Appendix 2.


## Using **VarargsInvoker**

Finally, we are at THAT moment when we can finally use the **VarargsInvoker**!
I'll skip a part where we create a linker, symbol lookups, and so on (they remain the same from Part 1),
but still, there are two topics to cover:
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
If you'll look at **Varargs::variadicLayout** you'll notice that there are only three types that can be processed further: _JAVA_LONG_, _JAVA_DOUBLE_, and **MemoryAddress**.
The reason for such "limitation" is that the rest of the Java types like **String**, **Integer**, and so on, require a native memory allocation.
Therefore, for _printf_, I have a limited scope of types to **MemoryAddress** only as the code operates by **String** variables that require native memory segment allocation.

### Memory segment and address allocations

What I'm trying to showcase, in C, would look like:
<script style="width: 1px;max-width: 100%;min-width: 100%;overflow: hidden;" src="//onlinegdb.com/embed/js/OQciSeTLn?theme=dark"></script>

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
Allocates a native memory for each of them, but for varargs - it grabs addresses to comply with a Java version of the _printf_ function.

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

The good thing about **VarargsInvoker** implementation is it can be used in all possible scenarios:
a function with variadic only, named and variadic and, named only or no args.

I know that such solutions go beyond ordinary software development,
but things will get much and much simpler when you will introduce a new code tool to your portfolio - jextract!

That's what I'm going to talk about in the next Part 3!

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
