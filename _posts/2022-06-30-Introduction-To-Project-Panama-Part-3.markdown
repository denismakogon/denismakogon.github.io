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

Both [Part 1](https://denismakogon.github.io/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html) and [Part 2](https://inside.java/2022/06/27/introduction-to-project-panama-part-2/) show that Project Panama offers the framework (Foreign Function & Memory API) designed to build and invoke C native code.
Part 1 was mostly an introduction to components necessary to declare and invoke a native function using C ABI, while Part 2 goal was to dig deeper into the variadic function aspects of the representation in Java applications. This article will go even deeper into details of the implementation of the variadic function provided in Part 2 as well as aspects of the implementation offered by the Project Panama code generating tool -- [jextract](https://github.com/openjdk/jextract).

## Source code compiling

### C code

Java and C compilers act differently due to the nature of programming language. The `javac` compiler turns [source code into bytecode](https://en.wikibooks.org/wiki/Java_Programming/Compilation#:~:text=In%20Java%2C%20programs%20are%20not%20compiled%20into%20executable%20files%3B%20they%20are%20compiled%20into%20bytecode) executed by the JVM in runtime while the C compiler transforms code text into a machine code.
It means that if the C compiler encounters a variadic function in code text it will have to create a new variant of that function, so the runtime will be able to execute a function with the respect to its parameters. Such a procedure is applied against all native variadic functions.

For variadic functions like the C _printf_, the compiler will create a specific version of the C _printf_ function that will correspond to the invocation signature.
For instance, the C _printf_ invocations down below will make the compiler create two different versions of the same function:
```cpp
#include <stdio.h>

int main() {
    printf("Hi! My name is %d\n", "Denis");
    printf("Hi! My name is %d. I'm %d years old\n", "Denis", 31);
}

```
for each invocation, the compiler will create a special variant of C _printf_:
```cpp
int(const char*, const char*);
int(const char*, const char*, const int);
```

In Java, the situation is completely different.
The [vararg](https://docs.oracle.com/javase/specs/jls/se18/html/jls-15.html#jls-15.12.2.4) is a runtime feature and every argument has a type (broader or narrower), while variadic arguments in C is a strictly compile-time feature.

### C code in Java

The infrastructure code for C _printf_ written in Java using Foreign Function & Memory API will explicitly show that a developer will have to do exactly what the C compiler does.

For each combination of variadic arguments, it's necessary to create a specialized variant of the **MethodHandle**. In situations like this, the **Linker** acts like the C compiler and the **FunctionDescriptor** as the **MethodType** holder:
```java
static final Linker linker = Linker.nativeLinker();
static final FunctionDescriptor puts$fd = FunctionDescriptor.of(JAVA_INT, ADDRESS);

// int(Addressable)
linker.downcallType(puts$fd);
```

as well as a hint to the Java compiler that explains how a native function would be invoked.
It appears to be that the Project Panama (Foreign Function & Memory API, in particular) mission is to let developers write C programs in Java by maintaining the same experience level,
but with a need to implement invocation infrastructure that is similar to what the C compiler does against C code:
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
}
```

The approach described in [Part 1](https://denismakogon.github.io/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html) and [Part 2](https://inside.java/2022/06/27/introduction-to-project-panama-part-2/) is based on the fact that a developer must create an instance of a linker, function descriptors and a method handles
that correspond to the particular C function (like the C _printf_ in [stdio.h](https://cplusplus.com/reference/cstdio/)).
The biggest concern is that a developer is not focused on building the application that uses the native function but on spending some time writing the infrastructure code for native functions.

This is one of the problems that Project Panama aims to solve is to provide tooling ([jextract](https://github.com/openjdk/jextract)) for generating the infrastructure code around the C native function that belongs to a particular C library.
So, the only responsibilities that the developer will have a focus on the invocations.

## jextract

Interesting fact, a new code generating tool is the first code tool that is a part of the OpenJDK but not a part of the distribution like any other JDK code tool.
There are a few reasons why it's not a part of the JDK distribution. First, not all Java developers work with native code so the `jextract` tool is not for everyone.
The second (and probably the most critical) reason -- it's big. Normally, the size of the JDK binary is less than 400Mb, the `jextract` tool adds 177Mb on top of it.
Indeed, it is that big. However, the `jextract` tool is not a typical tool that depends on the JDK internals.

The `jextract` tool has two dependencies:
* the JDK 19 or greater (Foreign Function & Memory API, in particular), 360Mb,
* [libclang](https://github.com/llvm/llvm-project) -- C interface to the Clang, 560Mb.

### libclang

As mentioned earlier, the `jextract` tool creates the infrastructure Java code for C native symbols based on the C-header file.
In `jextract`, `libclang` is responsible for:
- translating C source code into some intermediate state (also called as "front-end"),
- translate C source code intermedia state into machine code (also called "back-end", Clang uses LLVM for it).

But the most important thing is that Clang isn't just a compiler, it's also a library so-called "C interface to Clang".
So, the `jextract` tool uses [libclang](https://github.com/llvm/llvm-project) to parse C header files to extract whatever entity that has the native symbols (structs, typedefs, macros, vars, functions).

### JDK 19

`jextact` uses `java.lang.foreign` package that contain Foreign Function & Memory API.
But the most interesting thing is that `jextract` use Foreign Function & Memory API classes for two purposes: to work `libclang` native symbols and
model the infrastructure code for a native library it was instructed to create sources for.

## Using jextract

The best way to understand what the `jextract` tool does is to look at what it generates for the C stdio library.

### Installation

As mentioned before, `jextract` is the first standalone JDK code tool that is not a part of the OpenJDK distribution.
It means that to start working with the `jextract` tool it's necessary to obtain its binary available at [https://jdk.java.net/jextract](https://jdk.java.net/jextract).
The installation process is similar to the JDK installation because the `jextract` distribution package isn't just a single binary, but `jlink`-ed Java custom runtime.

So, download, unpack, and add to `$PATH`. Make sure the `jextract` tool is reachable:
```shell
jextract --version

jextract 19
JDK version 19-ea+23-1706
clang version 13.0.0
```

### Exploring `jextract`

Once the `jextract` tool is ready and functional, it's time to explore its capabilities. The purpose of `jextract` is to mechanically generate Java bindings from C native library headers.

To generate a Java source classes it's necessary to instruct the `jextract` tool as follows:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java /usr/include/stdio.h
```

Note: On macOS, C stdlib includes a folder (a path under the `-I` parameter) located here:
```shell
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include
```

The command above instruct `jextract` to generate Java source classes for `/usr/include/stdio.h` which base include folder is `/usr/include`,
resulting classes must be placed at `src/main/java` within `com.clang.stdlib.stdio` package.
Note that options like `-I` (include folder) is the same options presented in the `clang` compiler.
However, option `-l` (not mentioned in the command above) is a combination of two `gcc` flags: `-L <dir>` and `-l <library-name>`,
a value for this parameter could either be a library name or its absolute path.

### Exploring package content

As the result, the `jextract` tool will create a new Java package containing Java classes covering the C stdio library API mentioned in `stdio.h` header file:
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
To check what exactly `jextract` will generate for a header file it has the following parameter:
```shell
--dump-includes <file>         dump included symbols into specified file
```
This parameter will instruct the `jextract` tool to create a file containing all native symbols it could potentially extract from a header file.
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

Using the combination of these parameters it's possible to limit the amount of code `jextract` could generate.
For instance, the following command will create the infrastructure code only for C _printf_ native function:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java --include-function printf /usr/include/stdio.h
```

Filtering helps to avoid unnecessary Java source files as well as redundant functions, macros, structs, vars, and typedefs.
If `--dump-includes` is specified, `jextract` will only create this file, but will not generate sources.

So, the dump file is just a placeholder for `jextract` parameters, it can be supplied using a simple shell trick - `@<file>`:
```shell
jextract --source -t com.clang.stdlib.stdio -I /usr/include --output src/main/java  @dump.txt /usr/include/stdio.h
```

Note: It's essential to know what you are doing, there may be a dependency between components of a header file, like `stdio.h` `FILE` and `_sFILE` types where the `FILE` native symbol is an alias to the `sFILE` typedef.
Excluding one may lead to problems with others.

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
So, the only thing that a developer becomes responsible for is native memory allocation and function integration.

The amount of work necessary to be done before invoking a native function requires a lot less time and effort:
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

Filtering capability helps to be specific regarding what exactly has to be extracted from a header file.
However, the `jextract` tool doesn't track native symbol dependencies, for instance, a function
`int    vfscanf(FILE * __restrict __stream, const char * __restrict __format, va_list)` depends on `FILE` C struct.
But attempting to extract `vfscanf` would not make the `FILE` struct Java representation appear magically.
So, it's highly important to be explicit on what exactly you'd like to have as a result, the `jextract` tool is about a repeatable extraction process, where no magic occurs.
In other words, attempting to identify a dependency graph seems to be serving `jextract` users that don't want to extract the full thing,
but don't want to be specific enough to define their filtering. That's why filtering should be considered an advanced feature.

You see, with the `jextract` tool it all comes down to a question of how to invoke native functions, but not how to implement infrastructure around them.


## Code listing

You may find sources for this chapter [here](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-3).
<script src="https://emgithub.com/embed.js?target=https%3A%2F%2Fgithub.com%2Fdenismakogon%2Fopenjdk-project-samples%2Fblob%2Fmaster%2Fsrc%2Fmain%2Fjava%2Fcom%2Fjava_devrel%2Fsamples%2Fpanama%2Fpart_3%2FPrintf.java&style=github&showBorder=off&showLineNumbers=off&showFileMeta=on&showCopy=on"></script>
