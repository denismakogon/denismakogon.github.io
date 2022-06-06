---
layout: post
title:  'Introduction to Project Panama. Part 2: Simply implementing variadic functions.'
date:   2022-06-14
categories: openjdk panama
tags: ["openjdk", "panama"]
image_src_url: 'https://unsplash.com/photos/Wiwqd_8Rds8/download?ixid=MnwxMjA3fDB8MXxzZWFyY2h8OXx8cGFuYW1hfGVufDB8fHx8MTY1NDMyNzMwMg&force=true&w=800'
excerpt: 'This article explores problems of variadic functions and possible implementations of variadic functions using Foreign Function and Memory API.'
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

[Introduction to OpenJDK Project Panama. Part 1.]({{ '/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html' | relative_url }})

## Intro

This article explores a problem of variadic functions in Java and a simplified implementation of the variadic functions using Foreign Function and Memory API.

This article is into two pieces: one will cover simplified solutions to variadic functions problems, and another will do a deep dive into technical detail to explain the advanced solution to a problem of the variadic functions.

## "C variadic functions in Java" problem

### Part 1 recap

The purpose of [Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }})
was to make an introduction to the Foreign Function and Memory API.

As a practical task, in Part 1, I have shown you how to implement a downcall to the C _printf_ function using corresponding API classes:
* **SymbolLookup** - for searching native function address.
* **FunctionDescriptor** - for declaring a Java-based function descriptor containing a return value and argument layouts that correspond to the original signature in C.
* **Linker** - for creating a method handle for a native function from a descriptor and a native function address.
* **MemorySession** - for native memory segment allocation and de-allocation.

### Revisiting C _printf_ Java implementation

[Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }})
contains a simplified implementation of a downcall to the C _printf_ function in Java. That code did not implement most important feature - C variadic arguments for _printf_ function.
The absence of variadic arguments made C _printf_ implementation look like _PrintStream::println_ rather than _PrintStream::printf_ in Java standard library.

There is no _println_ in C standard library, so the functionality of _println_, as we got used to, can be replaced with _printf_:
```cpp
// the same as 'println("Hello World")' if it existed
printf("Hello World!\n");
```

The most significant difference between _PrintStream::println_ and _PrintStream::printf_ is the support of varargs in the last one, i.e., the _PrintStream::println_ is a simplified version of _PrintStream::printf_.

The "Hello World" application in Part 1 did not implement the complete variadic function contract of C _printf_. This article will close the technical dept and show how to develop downcalls to the C variadic functions using Java API.

### Variadic functions, variadic arguments

In C/C++, a variadic function, like _printf_, is a function type that takes a variable number of arguments indicated by the parameter of the form `...`
as a last one in the parameter list and must follow at least one named parameter:
```cpp
int     printf(const char * __restrict, ...);
```

At the moment of a call to the variadic function, it accepts variadic arguments. According to the variadic arguments definition, a length of variadic arguments could equal to zero or greater, however, there is a top limit of 127 variadic arguments per function call:
```cpp
printf("hello world");
printf(" I'm a=%s, I'm b=%s", a, b);
```

In C/C++, variadic functions have no type defined at a function declaration. In Java, a situation is the opposite:
```java
Object someMethod(Object...varargs);
```
The type of varargs is required. Often developers use broad varargs type definition using **Object** because most Java types inherit from it: any instance of an object which was type inherited from the **Object** will be accepted as a valid member of varargs at the runtime.

With all those facts, we need to answer how to implement C variadic arguments using Foreign Function and Memory API.

### Runtime representation of named and variadic arguments

In Part 1, the C _printf_ **FunctionDescriptor** had the following implementation:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
The C _printf_ is a variadic function, but there is no sign of variadic arguments in the descriptor.

According to C _printf_ definition, it has both named args (`const char * __restricted`) and variadic arguments (`...`), but the Java entry point to the C _printf_ (**MethodHandle::invoke**) accepts varargs only:
```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
```
i.e., there is no distinction between named args and varargs.

So, no matter what kind of combination of named and variadic arguments used to call a native function from the JVM, they must be enclosed into an array of type **Object[]** or passed one followed by another to remain in the order of defined by the **FunctionDescriptor**.

The **FunctionDescriptor** isn't just a representation of the C method signature but also declares the Java method type of the native function. Therefore, to perform a call native, the Java runtime must know what to call (a method handle) and how to invoke (based on the method type).

#### Native function descriptor and a method type

**FunctionDescriptor** is the key to a process of the native function method type validation (safe converting). It is the only entity the JVM relies on when it attempts to match a native method invocation method type and the descriptor method type as a reference value.

At the runtime, when the **MethodHandle::invoke** is called, the JVM attempts to perform safe type converting between
the method type (see [MethodType](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodType.html) derived from a function descriptor
```java
//                named arg      ret-value
//                <ADDRESS>      <JAVA_INT>
MethodHandle  (  Addressable  )     int
```
and a method type created from the arguments passed to the **MethodHandle::invoke** (both named and variadic).
In the case of the C _printf_, the JVM checks if it can convert safely whatever was submitted to the **MethodHandle::invoke**
as named and variadic arguments to a method type derived from Java implementation of the C _printf_ function descriptor.

So in Part 1, the **MethodHandle::invoke** was called with a Java string enclosed into a memory segment (**MemorySegment**):
```java
MemorySegment cString = memorySession.allocateUtf8String(str + "\n");
int res = (int) printfMethodHandle.invoke(cString);
```
the JVM created a method type from varargs (containing **MemorySegment**), then attempted to safely convert that it to a method type derived from the function descriptor:
```java
( MethodHandle(Addressable)int ) MethodHandle(MemorySegment)int
```

Note: Such type converting procedure would be successful because the **MemorySegment** class implements the [Addressable interface](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Addressable.html).

After spending some time investigating the JVM behavior, it appeared to be that it doesn't make any difference between named and variadic arguments. To showcase it, let's look at two varargs implementations.

## Implementing C variadic functions in Java

Out of the box, Foreign Function and Memory API offer a method for the explicit variadic arguments definition:
```java
FunctionDescriptor::asVariadic
```
Alongside that, I will offer you another implementation, the opposite of the explicit definition (implicit or dynamic).

### Explicit variadic argument types defined in advance

It's necessary to revisit the C _printf_ descriptor definition, again:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
as mentioned before, such declaration only covers a return value and named arg type.

According to the **FunctionDescriptor** [JavaDoc](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html#asVariadic(java.lang.foreign.MemoryLayout...)),
it is possible to explicitly define the exact variadic argument types within a function descriptor in advance:
```java
public FunctionDescriptor asVariadic(MemoryLayout... variadicLayouts)
```
it creates a specialized variadic function descriptor, by appending given variadic argument layouts to the function descriptor.

Let's say we want C _printf_ to accept _char * p_ and _int_ as variadic arguments:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
        );
FunctionDescriptor variadicFunction = function.asVariadic(ADDRESS, JAVA_INT);
```

Compared to C _printf_ definition, the Java-based function descriptor holds more details than C _printf_ version.
It says that a function that corresponds to a given descriptor returns _int_ value, accepts _char *_ as named arg
as well as **MANDATORY** variadic arguments of a specific length (two, for example) of types _char * p_ followed by _int_.
A call to a function of such description will have the following look:
```java
    var namedArg = memorySession.allocateUtf8String("My name is %s, age %d\n");
    var nameVararg = memorySession.allocateUtf8String("Denis");
    var ageVararg = 31;

    var ret = (int) printfHandle.invoke(namedArg, nameVararg, ageVararg);
```

What we know so far is that the JVM will check if it can convert the **MethodHandle::invoke** method type a method type derived from the `variadicFunction` descriptor:
```java
//      method type from the descriptor              method type from MethodHandle::invoke
( MethodType(Addressable, Addressable, int)int ) MethodHandle(MemorySegment,MemorySegment, int)int
```
Now it's clear that variadic argument types are a part of a method type which makes them mandatory parameters for a method handle invocation.
The beauty of variadic arguments in terms of software development is they could or may not be present at the invocation of a function.

So, the assumption was that the following code should've worked:
```java
(int) printfHandle.invoke(namedArg, nameVararg); // produces a method type (MemorySegment,MemorySegment,Void)int
(int) printfHandle.invoke(namedArg); // produces a method type (MemorySegment,Void,Void)int
```
Unfortunately, these both calls will fail with the corresponding exception -- **java.lang.invoke.WrongMethodTypeException**:
```java
Exception in thread "main" java.lang.RuntimeException: java.lang.invoke.WrongMethodTypeException: 
    cannot convert MethodHandle(Addressable,Addressable,int)int to (MemorySegment,MemorySegment,Void)int
```
and
```java
Exception in thread "main" java.lang.RuntimeException: java.lang.invoke.WrongMethodTypeException: 
    cannot convert MethodHandle(Addressable,Addressable,int)int to (MemorySegment,Void,Void)int
```

These invocations will produce exceptions because a method type implicitly created from a set of variadic arguments
will not match the expected type derived from the function descriptor used to create a method handle.

Here's where lies the biggest issue: it is hard to keep a combination of variadic arguments compliant with a function descriptor.
This is the illustration of potential problem:
```java
    private static final FunctionDescriptor printfDescriptor =
            FunctionDescriptor.of(JAVA_INT, ADDRESS);
    private static final FunctionDescriptor variadicPrintfDescriptor = printfDescriptor.asVariadic(ADDRESS, JAVA_INT);;

    private static int printf(String formatter, Object...varargs) {
        AtomicInteger res = new AtomicInteger(-1);
        symbolLookup.lookup("printf").ifPresent(addr -> {
            MethodHandle printfHandle = linker.downcallHandle(addr, variadicPrintfDescriptor);
            try {
                var cString = SegmentAllocator.implicitAllocator().allocateUtf8String(formatter);
                var args = new ArrayList<>(varargs.length + 1);
                Collections.addAll(args, List.of(cString));
                Collections.addAll(args, varargs);
                res.set(
                    (int) printfHandle.asSpreader(Object[].class, args.size()).invoke(args)
                );
            } catch (Throwable e) {
                throw new RuntimeException(e);
            }
        });
        return res.get();
    }
```

The ideal scenario is when it's possible to map Java varags to C variadic.
But what we see here is that a descriptor is defined in advance of a call, a descriptor is not aware of what the actual set of types Java varargs will hold.
Therefore, it's strictly required to impose Java varargs validation to fail-fast before performing the actual invocation.
A solution based on **FunctionDescriptor::asVariadic** indeed works and may serve its purpose, but the downside is the absence of flexibility, i.e., to Java varargs, say hello to named args in Java methods.

Good news! There's one more solution, that is meant to be a combination of flexibility and dynamic typing.
