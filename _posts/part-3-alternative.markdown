---
layout: post
title:  'Introduction to Project Panama. Part 3: jextract'
#date:   2022-06-21
categories: openjdk panama
tags: ["openjdk", "panama"]
image_src_url: 'https://unsplash.com/photos/Wiwqd_8Rds8/download?ixid=MnwxMjA3fDB8MXxzZWFyY2h8OXx8cGFuYW1hfGVufDB8fHx8MTY1NDMyNzMwMg&force=true&w=800'
excerpt: 'This article explores problems of variadic functions and possible implementations of variadic functions using Foreign Function and Memory API.'
---

## Introduction

Both [Part 1]() and [Part 2]() shown that Project Panama offers a great framework (Foreign Function & Memory API) designed to build and invoke C native code.
While Part 1 was mostly an introduction to components necessary to declare a native function in Java, the purpose of Part 2 was to dig dipper into the aspects of the variadic functions representation in Java. 
This article will go even deeper into details of the variadic functions implementation provided in Part 2 as well as aspects of the implementation offered by the Project Panama code generating tool -- [jextract](https://github.com/openjdk/jextract).

## Source code compiling

### C code

In Java, the compiler (`javac`) turns [source code into bytecode](https://en.wikibooks.org/wiki/Java_Programming/Compilation#:~:text=In%20Java%2C%20programs%20are%20not%20compiled%20into%20executable%20files%3B%20they%20are%20compiled%20into%20bytecode) while the C programs are compiled into a machine code.
It means that the C compiler must exactly tell to the runtime (the operating system) what and how a program will execute.

For functions accepting variadic arguments like the C _printf_, the compiler will have to create a specific version of the C _printf_ function that will correspond to the invocation signature.
For instance, the C _printf_ invocations down below will make the compiler to create two different version of the same function:
```java
#include <stdio.h>

int main() {
    printf("Hi! My name is %d\n", "Denis");
    printf("Hi! My name is %d. I'm %d years old\n", "Denis", 31);
}

```
as follows:
```java
int(const char*, const char*);
int(const char*, const char*, const int);
```

Comparing to Java methods that accepting the varargs, it's obvious that it does not relate to the C variadic function because varargs always have type (`Object... varargs`) definition.
Java varargs does not directly map to the C variadic arguments because the ability to invoke a _variable typed arity_ (vararg) method is a Java langauge feature described in [Java Language Specification](https://docs.oracle.com/javase/specs/jls/se18/html/jls-15.html#jls-15.12.2.4). 
It means that Java compiler will create a generic representation of a Java method that accepts the varargs in bytecode while the JVM will handle the invocation of such method using runtime-based arguments  (types, order and quantity).

### C code in Java

The takeaway is that Java runtime can handle dynamic varargs at the runtime, but it's not the thing for C because the variadic argument types, order, and quantity definition happen at compile time.
However, if we would compare Java varargs to the C variadic arguments written using the Foreign Function & Memory API it would be clear that a developer have to do exactly what the C compiler does against programs written in C.
In this context, think about **Linker** as the C compiler and the **FunctionDescriptor** as a hint to the compiler explaining how a native function would be invoked.
It appears to be that the Foreign Function & Memory API mission is to let developers write C programs in Java and compile these programs in Java runtime.

The reason why Part 2 was focused on a function descriptor declaration as on the most important component of the native function invocation.
But now we know that the **FunctionDescriptor** is the equivalent of the C source code that still has to be compiled in Java runtime. 
The sooner this happens, the less impact the compiling will cause on the application runtime. 
So, a function descriptor must be "compiled" into the method handle as early as possible:
```java
class PrintfImpls {
    static final Linker LINKER = Linker.nativeLinker();
    static final Addressable PRINTF_ADDR = LINKER.defaultLookup().lookup("printf").orElseThrow();

    static MethodHandle specializedPrintf(MemoryLayout... varargLayouts) {
        FunctionDescriptor specialized = PRINTF_BASE_TYPE.asVariadic(varargLayouts);
        return LINKER.downcallHandle(PRINTF_ADDR, specialized);
    }
    
    static final FunctionDescriptor PRINTF_BASE_TYPE = FunctionDescriptor.of(JAVA_INT, ADDRESS);

    public static final MethodHandle WithInt = specializedPrintf(JAVA_INT);
    public static final MethodHandle WithString = specializedPrintf(ADDRESS);
    public static final MethodHandle WithIntAndString = specializedPrintf(JAVA_INT, ADDRESS);
    
    public static void main(String... args) {
        
    }
}
```

Ideally, at JVM startup. At that stage the JIT compiler will try to inspect and pull apart constant method handles (defined as a static final field), 
to compile every call through a method handle like a call to any normal Java method causing the lowest performance impact on the application runtime.

Such behaviour aligns with the logic of the C compiler, but in Java, the "compiling" happen during the startup prior hitting the application entry point, 
so the application benefit from using "statically compiled" method handles (with the corresponding method type derived from the descriptor) instead of creating an instance of a method handle whenever it's necessary to invoke a native function.

## Generating Java classes for C functions

The approach described in Part 1 and Part 2 is based on the fact that a developer must create a function descriptor (and a method handle) 
that correspond to a certain C function mentioned in a C header file (like the C _printf_ in [stdio.h](https://cplusplus.com/reference/cstdio/)).
The problem is that a developer is not focused on building the application that uses native function, but spend some time writing the infrastructure code for native functions (linker, symbol lookup, method handles).

This is one of the problems that Project Panama code generating tools aim to solve -- to provide a tooling (`jextract`) for generating the infrastructure code around C native function that belong to a particular C library.
So, the only responsibilities that developer will have to take is a memory management and the invocation.

### jextract

Interesting fact, `jextract` is the first code tool that is a part of the OpenJDK but not a part of the distribution like any other JDK tools.
Basically, the `jextract` is a standalone project and a deliverable of the Project Panama under the OpenJDK umbrella.

There are a few reasons why it's not a part of the JDK distribution. First, not all Java developers are the JNI consumers, i.e., not all developers use `javac -h`, so the `jextract` is not for everyone.
The second and probably the most critical reason -- it's HUGE. Normally, size of the JDK is somewhat around 360Mb, the `jextract` adds on top around 177Mb.
Indeed, it is that big. However, the `jextract` is not a typical tool that depends on the JDK internals.
The `jextract` depends on:
* the JDK 19 or greater (Foreign Function & Memory API, in particular), 360Mb,
* [libclang](https://github.com/llvm/llvm-project) -- C interface to the Clang, 560Mb.

### libclang

As mentioned earlier, the `jextract` tool creates the infrastructure Java code for a C library.
If you aren't familiar with the Clang it is the C++ (and other C language family) compiler. It is responsible for:
- translating C source code into some intermediate state (also called as "front-end"),
- translate C source code intermedia state into machine code (also called as "back-end", Clang uses LLVM for it).

But the most important thing that Clang isn't just a compiler, it's also a library so-called "C interface to Clang". 
So, the `jextract` doesn't use its own type of parser for reading C header files, it uses [libclang](https://github.com/llvm/llvm-project) instead.

### JDK

Interesting that the `jextract` uses Foreign Function & Memory API not only to access data structures created by the `libclang` for the particular C header file,
but also to create a descriptor, a method handle for each C native function mentioned in a header file as well as a runtime helper class containing common utility functionality.

### Java sources for C stdio library

The best way to understand what the `jextract` does it's better to look at what it generates for the C stdio library.
To generate Java source it's necessary to instruct the `jextract` as follows:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java /usr/include/stdio.h
```

Note: On macOS, C stdlib include folder (a path under `-I` flag) located here: `/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include`

#### Package content

As the result, the `jextract` will create a brand-new Java package containing Java classes covering C stdio library API mentioned in `stdio.h` header file:
```shell
src/main/java/com/clang/stdlib/stdio
├── Constants$root.java
├── FILE.java
├── RuntimeHelper.java
├── __darwin_mbstate_t.java
├── __darwin_pthread_attr_t.java
├── __darwin_pthread_cond_t.java
├── __darwin_pthread_condattr_t.java
├── __darwin_pthread_handler_rec.java
├── __darwin_pthread_mutex_t.java
├── __darwin_pthread_mutexattr_t.java
├── __darwin_pthread_once_t.java
├── __darwin_pthread_rwlock_t.java
├── __darwin_pthread_rwlockattr_t.java
├── __mbstate_t.java
├── __sFILE.java
├── __sbuf.java
├── _opaque_pthread_attr_t.java
├── _opaque_pthread_cond_t.java
├── _opaque_pthread_condattr_t.java
├── _opaque_pthread_mutex_t.java
├── _opaque_pthread_mutexattr_t.java
├── _opaque_pthread_once_t.java
├── _opaque_pthread_rwlock_t.java
├── _opaque_pthread_rwlockattr_t.java
├── _opaque_pthread_t.java
├── constants$0.java
...
├── constants$17.java
├── funopen$x0.java
├── funopen$x1.java
├── funopen$x2.java
├── funopen$x3.java
└── stdio_h.java

0 directories, 48 files
```

#### Public classes

The entry point to the library is a Java class file named as the header file (by default) - `stdio_h.java`. It contains a public class that represents the C stdio API.
Often there would only one public API. However, if a header file contain too many native functions then it's possible that the `jextract` will split generated sources into multiple API classes connected to each other through the inheritance, but the still there would be only one public API class.

There also could be more public classes that represent C structs, each struct will have its own file. So, in case of the C stdio, there's one more public class - `FILE.java`:
```java
// Generated by jextract

package com.java_devrel.samples.stdlib.stdio;

public class FILE extends __sFILE {

}
```

Interesting that the C `FILE` structure isn't a part of the `stdio.h` header file. It means that the `jextract` recursively digs into a header 
file until it find all components that a top-level header file depends on. So, in the case of C `FILE`, the `jextract` loops through header files 
mentioned in the C `stdio.h` in a search of the `FILE` native symbol. But it appears to be that C `FILE` doesn't even exist as a C structure, 
it's a [typedef with an alias](https://en.cppreference.com/w/cpp/language/typedef) in `_stdio.h` header file included at the top of the `stdio.h`:
```java
typedef	struct __sFILE {
        ...
} FILE;
```

In Java, there are no aliases, so the equivalent of a typedef with an alias is a class extending inheritance model.
So, the `jextract` creates a public class named as a typedef alias (`FILE.class`) and another private class with a name given in typedef construction (`sFILE`).


**Important**: No matter what kind of C header file used to generate Java sources for a C library, there always would be two types of public API classes: structs, enums and so on, and an API class that represents a C library API mentioned in a header file.

#### Private classes

As mentioned before, to make a call to a native function a developer must have a function descriptor and a method handle created from it.
So, files named as `constants$XXX.java` contain all necessary function descriptors that a header file contain and method handles, all objects are effectively final and static which guarantees the best performance as mentioned before.

The last but not the least important private class is `RuntimeHelper.class`, this class is responsible for creating downcall handles for C native function. It's a general-purpose class that consists of three important components:
- `MethodHandle downcallHandle(String name, FunctionDescriptor fdesc)` -- creates a downcall handle for C standard functions,
- `MethodHandle downcallHandleVariadic(String name, FunctionDescriptor fdesc)` -- creates a downcall handle for C variadic functions,
- `class VarargsInvoker` -- handles C variadic function invocations.

Moreover, the last two are connected to each other, let's look at them closely.

### Handling C variadic function in the `jextract` way

As mentioned in the beginning of this article, the most effective way to implement C variadic function using 
Foreign Function & Memory API is to make create a static descriptor and a method handle for each potential variadic argument combination:
```java
    public static final MethodHandle WithInt = specializedPrintf(JAVA_INT);
    public static final MethodHandle WithString = specializedPrintf(ADDRESS);
    public static final MethodHandle WithIntAndString = specializedPrintf(JAVA_INT, ADDRESS);
```

At the same time, the `jextract` offers another way to do so. The overall idea can be expressed as:

> The `RuntimeHelper::VarargsInvoker` imposes a proxy between a native function consumer and the actual downcall handle created through a **Linker**.

A proxy (`class VarargsInvoker`) consists of two components: a method handle constructor (`VarargsInvoker::make`) and the invocation handler (`Varargs::invoke`).

#### `VarargsInvoker::make`

By saying proxy, I mean there is a Java method standing between the actual downcall handle and its consumer.
So, to preserve the same experience as with C standard functions in Java, `VarargsInvoker::make` creates a custom variant of the `VarargsInvoker::invoke` an array-collecting method handle
which method type consists of a function descriptor and **Object[].class** as placeholder for potential variadic arguments.

Note: Array-collecting method handle instructs the runtime that whenever Java method is invoked through a method handle, all its parameters will be grouped into array of a particular type.
The `VarargsInvoker::make` defines array type as the **Object[].class**.

In case of the C _printf_, the `VarargsInvoker::make` will create a method handle white method type is:
```java
(Addressable, Object[].class)int
```

This method type indicates that Java method must be invoked with two parameters -- with an instance of the `Addressable` and the **Object[]** array.

You see, the goal of the `VarargsInvoker::make` is to create a method handle which method type match to C function 
signature where an array plays a role the variadic arguments:
```java
// int print( const char* __restricted,      ...      );
             (    Addressable,         Object[].class )int
```

That's why the base function descriptor for the C _printf_ contain named argument layout only, to make it possible to create a new variant of a function descriptor that will contain the variadic argument layouts.
But this time just before a downcall handle invocation, meaning that it would be necessary to "compile" a new version of the C _printf_ that corresponds to the invocation parameters.
All of that will happen in a runtime which makes it hard for the JVM to optimize this code.

Such approach is flexible, indeed. However, it's not effective in terms of the performance which makes the solution described in Part 2 more efficient, but not flexible. 

#### `VarargsInvoker::invoke`

While the `VarargsInvoker::make` is responsible for creating a method handle, the `VarargsInvoker::invoke` is the component responsible for the actual invocation.
In a runtime, a native function consumer will invoke it through an array-collecting method handle provided by the `VarargsInvoker::make` 
which means that the `VarargsInvoker::invoke` will accept all parameters in a form array. It will contain named arguments followed by another array containing variadic arguments:
```java
Object[<namedArg1>, ... , <namedArgN>, Object[<varArg1>, ... , <varArgN>]]
```

Having this array, the `VarargsInvoker::invoke` must a few tasks:
- guess the layouts for all arguments;
- check if named argument layouts match ot a function descriptor provided to `VarargsInvoker::make`;
- form a new function descriptor (that will contain all variadic argument layouts) and a new downcall handle using a **Linker**;
- flatten parameters array and do the invocation.

give an example!