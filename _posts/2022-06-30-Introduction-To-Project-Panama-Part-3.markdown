---
layout: post
title: 'Introduction to Project Panama. Part 3: jextract'
date: 2022-06-30
categories: openjdk panama
tags: ["openjdk", "panama"]
image_src_url: 'https://unsplash.com/photos/Wiwqd_8Rds8/download?ixid=MnwxMjA3fDB8MXxzZWFyY2h8OXx8cGFuYW1hfGVufDB8fHx8MTY1NDMyNzMwMg&force=true&w=800'
excerpt: 'This article introduces the Foreign Function & Memory API using a simple Java-based "Hello World" application invoking some C native code.'
language: English
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})

Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
    

## Introduction

Both [Part 1](https://denismakogon.github.io/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html) and [Part 2](https://inside.java/2022/06/27/introduction-to-project-panama-part-2/) show that Project Panama offers a great framework (Foreign Function & Memory API) designed to build and invoke C native code. 
While Part 1 was mostly an introduction to components necessary to declare a native function in Java, the purpose of Part 2 was to dig deeper into the aspects 
of the representation of the variadic function in Java. This article will go even deeper into details of the implementation of the variadic function provided 
in Part 2 as well as aspects of the implementation offered by the Project Panama code generating tool -- [jextract](https://github.com/openjdk/jextract).

## Source code compiling

### C code

In Java, the compiler (`javac`) turns [source code into bytecode](https://en.wikibooks.org/wiki/Java_Programming/Compilation#:~:text=In%20Java%2C%20programs%20are%20not%20compiled%20into%20executable%20files%3B%20they%20are%20compiled%20into%20bytecode) while the C programs are compiled into a machine code.
It means that the C compiler must exactly tell the runtime (the operating system) what and how a program will execute.

For functions accepting variadic arguments like the C _printf_, the compiler will have to create a specific version of the C _printf_ function that will correspond to the invocation signature.
For instance, the C _printf_ invocations down below will make the compiler create two different versions of the same function:
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

Compared to Java methods that accept the varargs, it's obvious that it does not relate to the C variadic function because varargs always have a type (`Object... varargs`) definition.
Java varargs do not directly map to the C variadic arguments because the ability to invoke a _variable typed arity_ (vararg) method is a Java language feature described in [Java Language Specification](https://docs.oracle.com/javase/specs/jls/se18/html/jls-15.html#jls-15.12.2.4).
It means that the Java compiler will create a generic representation of a Java method that accepts the varargs in bytecode while the JVM will handle the invocation of such method using runtime-based arguments  (types, order, and quantity).

### C code in Java

The takeaway is that the Java runtime can handle dynamic varargs at the runtime. However, it's not the thing for C because the variadic argument types, order, and quantity definition happen at compile time.
However, if we would compare Java varargs to the C variadic arguments written using the Foreign Function & Memory API it would be clear that a developer has to do exactly what the C compiler does against programs written in C.
In this context, think about **Linker** as the C compiler and the **FunctionDescriptor** as a hint to the compiler explaining how a native function would be invoked.
It appears to be that the Foreign Function & Memory API's mission is to let developers write C programs in Java and compile these programs in Java runtime.

The reason why Part 2 was focused on a function descriptor declaration as to the most important component of the native function invocation.
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

Ideally, at JVM startup. At that stage, the JIT compiler will try to inspect and pull apart constant method handles (defined as a static final field),
to compile every call through a method handled like a call to any normal Java method causing the lowest performance impact on the application runtime.

Such behavior aligns with the logic of the C compiler, but in Java, the "compiling" happens during the startup before hitting the application entry point,
so the application benefit from using "statically compiled" method handles (with the corresponding method type derived from the descriptor) instead of creating an instance of a method handle whenever it's necessary to invoke a native function.

The approach described in Part 1 and Part 2 is based on the fact that a developer must create a function descriptor (and a method handle)
that correspond to a certain C function mentioned in a C header file (like the C _printf_ in [stdio.h](https://cplusplus.com/reference/cstdio/)).
The problem is that a developer is not focused on building the application that uses the native function, but spends some time writing the infrastructure code for native functions (linker, symbol lookup, method handles).

This is one of the problems that Project Panama code generating tools aim to solve -- to provide tooling (`jextract`) for generating the infrastructure code around the C native function that belongs to a particular C library.
So, the only responsibilities that the developer will have to take are memory management and invocation.

## Prerequisites

Interesting fact, `jextract` is the first code tool that is a part of the OpenJDK but not a part of the distribution like any other JDK tool.
The `jextract` tool is a standalone project and a deliverable of Project Panama under the OpenJDK umbrella.

There are a few reasons why it's not a part of the JDK distribution. First, not all Java developers are the JNI consumers, i.e., not all developers use `javac -h`, so the `jextract` tool is not for everyone.
The second and probably the most critical reason -- it's HUGE. Normally, the size of the JDK is somewhat around 360Mb, the `jextract` tool adds on top around 177Mb.
Indeed, it is that big. However, the `jextract` tool is not a typical tool that depends on the JDK internals.
the `jextract` tool  depends on:
* the JDK 19 or greater (Foreign Function & Memory API, in particular), 360Mb,
* [libclang](https://github.com/llvm/llvm-project) -- C interface to the Clang, 560Mb.

### libclang

As mentioned earlier, the `jextract` tool creates the infrastructure Java code for a C library.
If you aren't familiar with the Clang it is the C++ (and other C language family) compiler. It is responsible for:
- translating C source code into some intermediate state (also called as "front-end"),
- translate C source code intermedia state into machine code (also called "back-end", Clang uses LLVM for it).

But the most important thing is that Clang isn't just a compiler, it's also a library so-called "C interface to Clang".
So, the `jextract` tool doesn't use its type of parser for reading C header files, it uses [libclang](https://github.com/llvm/llvm-project) instead.

### JDK 19

Interesting that the `jextract` tool  uses Foreign Function & Memory API not only to access data structures created by the `libclang` for the particular C header file,
but also to create a descriptor, a method handle for each C native function mentioned in a header file as well as a runtime helper class containing common utility functionality.

## Using jextract

The best way to understand what the `jextract` tool does it's better to look at what it generates for the C stdio library.

### Building from source

As mentioned before, `jextract` is the first standalone JDK code tool that is not a part of the OpenJDK distribution.
It means that to start working with the `jextract` tool it's necessary to obtain its binary.

As of now, there are no binary releases available at the `jextract` GitHub repo. So, it's necessary to build one from sources.
So, to build `jextract` from source code, it's necessary to install the JDK 19 EA build and set it as the default version:
```shell
sdk install java 19.ea.28-open
```

Obtain Clang build (version 9 or greater):
```shell
mkdir ${HOME}/clang/llvm
wget -O clang+llvm-13.0.0-x86_64-apple-darwin.tar.xz https://github.com/llvm/llvm-project/releases/download/llvmorg-13.0.0/clang+llvm-13.0.0-x86_64-apple-darwin.tar.xz
tar --strip-components=1 -xvf clang+llvm-13.0.0-x86_64-apple-darwin.tar.xz -C ${HOME}/clang/llvm
```

Clone `jextract` source code and run the build it:
```shell
git clone https://github.com/openjdk/jextract.git && cd jextract

sh ./gradlew -Pjdk19_home=${HOME}/.sdkman/candidates/java/19.ea.28-open -Pllvm_home=${HOME}/clang/llvm clean verify
```

Note: Clang distribution archives are both platform and CPU architecture-specific, [here](https://github.com/llvm/llvm-project/releases/tag/llvmorg-13.0.0) you may find AMD64 and AARCH64 builds of Linux and Windows.

As the result, `gradlew` will create a Java custom runtime:
```shell
build/jextract
├── bin
├── conf
├── include
├── legal
├── lib
└── release
```
which is almost the same as the one provided through `-Djdk19_home` except few things, it was built with a new Java module containing the `jextract` functionality and a shell script:
```shell
$ file build/jmods/org.openjdk.jextract.jmod 

build/jmods/org.openjdk.jextract.jmod: Java jmod module version 1.0
```
```shell
$ file build/jextract/bin/jextract  && cat build/jextract/bin/jextract 

build/jextract/bin/jextract: POSIX shell script text executable, ASCII text

#!/bin/sh
JLINK_VM_OPTIONS=
DIR=`dirname $0`
$DIR/java $JLINK_VM_OPTIONS -m org.openjdk.jextract/org.openjdk.jextract.JextractTool "$@"
```

The last step to make the `jextract` tool consumable outside the build folder it's necessary to extend the system `$PATH` definition with `build/jextract/bin` folder:
```shell
export PATH=$PATH:$PWD/build/jextract/bin
```
Make sure the `jextract` tool is reachable:
```shell
jextract --version

jextract 19-ea
JDK version 19-ea+28-2110
clang version 13.0.1
```

### Exploring `jextract`

Once the `jextract` tool is ready and functional, it's time to explore its capabilities.
The main purpose of `jextract` is to mechanically generate Java bindings from C native library headers.

To generate Java source it's necessary to instruct the `jextract` tool as follows:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java /usr/include/stdio.h
```

Note: On macOS, C stdlib include folder (a path under `-I` option) located here:
```shell
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include
```

This command tells to `jextract` to generate Java source classes for `/usr/include/stdio.h` which base include folder is `/usr/include`, resulting classes must be placed at `src/main/java` within `com.clang.stdlib.stdio` package.
Note that options like `-I` (include folder) is the same options presented in the `clang` compiler. However, option `-l` (not mentioned in the command above) is a combination of two `gcc` flags: `-L <dir>` and `-l <library-name>`.
Because the `jextract` `-l` option provides a certain level of flexibility, a value for this option could be either a library name or its absolute path.

### Exploring package content

As the result, the `jextract` tool will create a new Java package containing Java classes covering C stdio library API mentioned in `stdio.h` header file:
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

### Advanced usage

As you see, the `jextract` tool generates a lot of Java sources for the C `stdio.h` header file. Not all of those header files may be required to execute the particular C function.
So, `jextract` has the option to check what kind of content would be created:
```shell
--dump-includes <file>         dump included symbols into specified file
```
This option will instruct the `jextract` tool to create a file containing all native symbols it could potentially generate.
A newly created file will contain a combination of the following option values:
```shell
--header-class-name <name>         
--include-function <name>
--include-enum <name>
--include-macro <name>
--include-struct <name>
--include-typedef <name>
--include-union <name>
--include-var <name>
```

For instance:
```shell
--include-macro MAC_OS_VERSION_12_0
...
--include-typedef FILE
...
--include-var __stdoutp
...
--include-function printf
```

Filtering helps to avoid unnecessary Java source files as well as redundant functions, macros, structs, vars, and typedefs.
If `--dump-includes` is specified, `jextract` will only create this file, but will not generate sources.

To use this file simply addi it to a list of parameters:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java  @dump.txt /usr/include/stdio.h
```

Note: It's essential to know what you are doing, there may be a dependency between components of a header file, like `stdio.h` `FILE` and `_sFILE` (where's `FILE` is an alias to the `sFILE` typedef).
Excluding one may lead to problems with others.


## Straight to the point

The good news that the C _printf_ has no internal dependencies inside `stdio.h`, so in order to create the infrastructure code for it,
the `jextract` tool must be instructed as follows:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java --include-function printf /usr/include/stdio.h
```

As the result, the `jextract` tool will create a smaller version of a package mentioned above containing exactly what it was told to export - the C _printf_ function only.
```tree
src/main/java/com/clang/stdlib/stdio
├── Constants$root.java
├── RuntimeHelper.java
├── constants$0.java
└── stdio_h.java

0 directories, 4 files
```

Generated Java sources will contain the bare minimum code necessary to invoke the C _printf_ function, a descriptor, and a method handle:
```java
class constants$0 {

    static final FunctionDescriptor printf$FUNC = FunctionDescriptor.of(Constants$root.C_INT$LAYOUT,
        Constants$root.C_POINTER$LAYOUT
    );
    static final MethodHandle printf$MH = RuntimeHelper.downcallHandleVariadic(
        "printf",
        constants$0.printf$FUNC
    );
}
```
as well as public API method:
```java
public class stdio_h  {

    /* package-private */ stdio_h() {}

    public static MethodHandle printf$MH() {
        return RuntimeHelper.requireNonNull(constants$0.printf$MH,"printf");
    }
    public static int printf ( Addressable x0, Object... x1) {
        var mh$ = printf$MH();
        try {
            return (int)mh$.invokeExact(x0, x1);
        } catch (Throwable ex$) {
            throw new AssertionError("should not reach here", ex$);
        }
    }
}
```

With the `jextract` tool the path from identifying the C native library necessary to the actual integration of it into
Java application is much shorter compared to the approach described in Part 1 and Part 2.
So, the only thing that a developer becomes responsible for is a native memory allocation and function integration.

The amount of work necessary to be done prior invoking a native function requires a lot less time and efforts:
```java
import java.lang.foreign.MemorySession;
import java.lang.foreign.SegmentAllocator;
import com.java_devrel.samples.stdlib.stdio.stdio_h;

public class Printf {
    public static void main(String[] args) {
        try (var memorySession = MemorySession.openConfined()) {
            stdio_h.printf(memorySession.allocateUtf8String(
                    "Welcome from the other side!\n"));
        }
    }
}
```

You see, with the `jextract` tool, it all comes down to a question of how to invoke native functions, but not how to implement infrastructure around them.
At the same time, `jextract` provides enough capabilities to turn C headers into Java source code.

Filtering capability helps to be specific regarding what exactly has to be extracted from a header file.
However, the `jextract` tool doesn't track native symbol dependencies, for instance, a function
`int    vfscanf(FILE * __restrict __stream, const char * __restrict __format, va_list)` depends on `FILE` C struct.
But attempting to extract `vfscanf` would not make the `FILE` struct Java representation appear magically.
So, it's highly important to be explicit on what exactly you'd like to have as a result, the `jextract` tool is about a repeatable extraction process, where no magic occurs.
In other words, attempting to identify a dependency graph seems to be serving `jextract` users that don't want to extract the full thing,
but don't want to be specific enough to define their filtering. That's why filtering should be considered an advanced feature.

### To be continued

In the next post, we'll explore a package structure as well as Java classes content provided by `jextract`.


## Code listing

You may find sources for this chapter [here](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-3).
<script src="https://emgithub.com/embed.js?target=https%3A%2F%2Fgithub.com%2Fdenismakogon%2Fopenjdk-project-samples%2Fblob%2Fmaster%2Fsrc%2Fmain%2Fjava%2Fcom%2Fjava_devrel%2Fsamples%2Fpanama%2Fpart_3%2FPrintf.java&style=github&showBorder=off&showLineNumbers=off&showFileMeta=on&showCopy=on"></script>
