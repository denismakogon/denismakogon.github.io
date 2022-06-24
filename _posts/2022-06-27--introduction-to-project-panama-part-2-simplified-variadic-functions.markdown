---
layout: post
title:  'Introduction to Project Panama. Part 2: Simply implementing variadic functions.'
date:   2022-06-27
categories: openjdk panama
tags: ["openjdk", "panama"]
image_src_url: 'https://unsplash.com/photos/-st0dxfDyM0/download?ixid=MnwxMjA3fDB8MXxzZWFyY2h8MzR8fHBhbmFtYXxlbnwwfHx8fDE2NTU4OTQ2ODg&force=true&w=800'
excerpt: 'This article explores problems of variadic functions and possible implementations of variadic functions using Foreign Function & Memory API.'
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez--st0dxfDyM0-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


## Introduction

This article explores Java native variadic functions using the Foreign Function & Memory API (Project Panama).

## C Variadic Functions in Java

### Part 1 recap

The purpose of [Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }})
was to introduce the Foreign Function & Memory API. As a practical task, Part 1 shows you how to implement a downcall to the C _printf_ function using corresponding API classes:
* **SymbolLookup** -- Looking up a native function memory address.
* **FunctionDescriptor** -- Declaring a Java-based function descriptor containing a return value and argument layouts that correspond to the original signature in C.
* **Linker** -- Creating a method handle wrapping the native function from a function descriptor and its native function address.
* **MemorySession** -- Allocating and de-allocating native memory segments.

### Revisiting C _printf_ Java implementation

[Part 1]({{ '/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html' | relative_url }}) contains a simplified implementation of a downcall to the C _printf_ function in Java.
But that code did not implement a key feature -- C variadic arguments for the _printf_ function.
The absence of variadic arguments made the C _printf_ implementation looks like the _PrintStream::println_ method rather than the _PrintStream::printf_ method from the Java standard library.

The key difference between _PrintStream::println_ and _PrintStream::printf_ is the support of varargs in the latter, meaning the _PrintStream::println_ is a simplified version of _PrintStream::printf_.

For the sake of simplicity, the “Hello World” example in Part 1 did not implement the variadic contract for the C _printf_ function.
This article will explain how to invoke from Java a native C variadic function (downcall) using the Foreign Function & Memory API.

### Variadic functions, variadic arguments

In C/C++, a [variadic function](https://en.cppreference.com/w/c/language/variadic) (ex. _printf_) is a function type that accepts a variable number of arguments.
The list of parameters of a variadic function should always start with at least one named parameter and should always end with the `...` parameter:
```c
int     printf(const char * __restrict, ...);
```

At the runtime, a function can accept no more than 127 variadic arguments per invocation:
```c
printf("hello world");
printf(" I'm a=%s, I'm b=%s", a, b);
```

In C/C++, variadic arguments don’t have any type associated in the signature. In Java, it is the opposite -- the type of varargs is required:
```java
Object someMethod(Object... varargs);
```

Java developers often use the broad varargs type definition using **java.lang.Object** because all Java types are inherited from it.
With all those facts described above, we now need to understand how to implement C variadic arguments using the Foreign Function & Memory API.

### Runtime representation of named and variadic arguments

In Part 1, the C _printf_ **FunctionDescriptor** had the following implementation:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
        );
```
The C _printf_ is a variadic function, but there is no indication of any variadic arguments in the function descriptor.

According to the C _printf_ definition, it has both named (positional) arguments (`const char * __restricted`) and variadic arguments (`...`) while the Java entry point of the C _printf_ function (**MethodHandle::invoke**) only accepts varargs arguments (there is no distinction between named arguments and varargs):
```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
```

So, no matter what kind of combination of named and variadic arguments is provided to call a native function, they must be passed either as array of **java.lang.Object** or as varargs:
```java
var allArgs = new Object[] {namedArg1, ..., NamedArgN, varArg1, ..., varArgN};
        methodHandle.asSpreader(Object[].class, allArgs.length).invoke(allArgs);
```

or passed as varargs with named arguments first, followed by the variadic arguments, to preserve the order and comply with the **FunctionDescriptor** definition:
```java
methodHandle.invoke(namedArg1, ..., NamedArgN, varArg1, ..., varArgN);
```

In addition to the C method signature (a return value, names arguments order, and variadic arguments), a **FunctionDescriptor** also declares the Java method type of that same native function.
Therefore, to perform a native call, the Java runtime must know the method handle and how to call the native code based on the method type.

#### Native function descriptor and method type

The **FunctionDescriptor** is key to invoking the method type validation of a native function (safe type converting) and is the only entity the JVM relies on when attempting to match the **MethodHandle::invoke** method type to the function descriptor method type.

At runtime, **MethodHandle::invoke** is called, and the JVM attempts safe type converting between the [MethodType](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodType.html) derived from a function descriptor
```java
//                named arg      ret-value
//                <ADDRESS>      <JAVA_INT>
MethodHandle  (  Addressable  )     int
```
and a method type created from the arguments passed to the **MethodHandle::invoke** (both named and variadic). The JVM will recognize type mismatch as an exceptional situation.

In the case of the C _printf_, the JVM will check if it can safely convert a combination of named and variadic arguments submitted to the **MethodHandle::invoke** to a method type derived from the Java implementation of the C _printf_ function descriptor.

In Part 1, the **MethodHandle::invoke** was called with a **java.lang.String** object enclosed into a memory segment (**MemorySegment**):
```java
MemorySegment cString = memorySession.allocateUtf8String(str + "\n");
        int res = (int) printfMethodHandle.invoke(cString);
```

During the invocation, the JVM will create a method type using the arguments (`MethodHandle(MemorySegment)int`), the JVM will then attempt to convert it to a method type derived from the function descriptor:
```java
//    <descriptor method type>       <invoke method type>
( MethodHandle(Addressable)int ) MethodHandle(MemorySegment)int
```

Note: Such type converting procedure will succeed because the **MemorySegment** interface extends the [Addressable interface](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Addressable.html).

The key takeaway is that the JVM is leveraging the **FunctionDescriptor** return value and the argument layouts to create a method type.
It means that the argument types, order, and quantity as well as the return value type validation will be enforced by the JVM during the invocation of a native function:
```java
var emptyDescriptor = FunctionDescriptor.of(JAVA_INT);
System.out.println(Linker.downcallType(emptyDescriptor));

var descriptorWithNamedArg = emptyDescriptor.appendArgumentLayouts(ADDRESS);
System.out.println(Linker.downcallType(descriptorWithNamedArg));
```
```shell
()int
(Addressable)int
```

## Implementing C variadic functions using the Foreign Function & Memory API

The new Foreign Function & Memory API offers a method for the [explicit variadic arguments definition](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html#asVariadic(java.lang.foreign.MemoryLayout...)):
```java
public FunctionDescriptor asVariadic(MemoryLayout... variadicLayouts)
```

The argument layouts provided to the **FunctionDescriptor** builder through the **FunctionDescriptor::asVariadic** 
will become a mandatory part of a method type following the named argument types:
```java
var descriptorWithNamedAndVariadicArg = descriptorWithNamedArg
        .asVariadic(ADDRESS, JAVA_INT);
System.out.println(Linker.downcallType(descriptorWithNamedAndVariadicArg));
```
```shell
(Addressable,Addressable,int)int
```

Let's say we want to implement the following C _printf_ downcall:
```c
printf("My name is %s, age %d\n",  "Denis", 31)
```

The C _printf_ will accept _char * p_ and _int_ as variadic arguments, so the function descriptor must declare
them explicitly in the corresponding order:
```java
FunctionDescriptor descriptorWithNamedAndVariadicArg = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
).asVariadic(ADDRESS.withBitAlignment(64), JAVA_INT.withBitAlignment(32));
```

Compared to the C _printf_ definition, the `descriptorWithNamedAndVariadicArg` descriptor holds additional details regarding
variadic arguments (types, order, and quantity). A call to a function defined by such description will look like:
```java
    var namedArg = memorySession.allocateUtf8String("My name is %s, age %d\n");
    var nameVararg = memorySession.allocateUtf8String("Denis");
    var ageVararg = 31;

    var ret = (int) printfHandle.invoke(namedArg, nameVararg, ageVararg);
```

The JVM will check if it can safely convert the invocation method type to a method type derived from the `function` descriptor:
```java
System.out.println(Linker.downcallType(descriptorWithNamedAndVariadicArg));
```
```shell
(Addressable,Addressable,int)int
```

The attractiveness of variadic arguments is based on their variadic nature. However, using them can involve more work for the Java developer,
like creating a new descriptor for each possible argument combination and a related method handle.
So, the following code will fail:
```java
(int) printfHandle.invoke(namedArg, nameVararg); // method type: (MemorySegment,MemorySegment,Void)int
(int) printfHandle.invoke(namedArg); // method type: (MemorySegment,Void,Void)int
```

Each call to the C _printf_ will fail with the corresponding exception:
```java
Exception in thread "main" java.lang.RuntimeException: java.lang.invoke.WrongMethodTypeException: 
    cannot convert MethodHandle(Addressable,Addressable,int)int to (MemorySegment,MemorySegment,Void)int
```

and
```java
Exception in thread "main" java.lang.RuntimeException: java.lang.invoke.WrongMethodTypeException: 
    cannot convert MethodHandle(Addressable,Addressable,int)int to (MemorySegment,Void,Void)int
```

These invocations will fail and raise exceptions because of the mismatch between the actual method type and the expected type.
The main challenge for Java developers is to maintain the combination of variadic arguments compliant with the function descriptor.
Every new named and variadic arguments combination requires a new function descriptor and eventually a new related method handle.

There is a strong dependency between invocation parameters, function descriptors, and method handles. An invocation **must** always comply with the function descriptor.
If not, it will lead to the exception.

## Flexibility and performance

A solution based on the variadic arguments declaration in advance will work in many cases because not all variadic functions are like the C _printf_ function.
At the same time, native functions like C _printf_ can handle a great variety of variadic arguments.
So, the goal is to preserve the same level of flexibility during the invocation to functions like C _printf_.

Unfortunately, the solution described in this article lacks the flexibility that variadic arguments offer. Declaring variadic arguments in advance turns them into an integral part of the method type, meaning those arguments become mandatory at invocation.

Another concern is the performance impact as there are too many invocation-specific components. For example, the runtime would have to create a new instance of a method handle for every variadic arguments combination.

Think about the **Linker** as the compiler. It must know what kind of method handles it needs to create before the invocation.
The sooner this happens, ideally at JVM startup, the less impact it will cause on the application runtime.
So, for performance reasons, a method handle for each variadic argument combination should be stored inside a static final field and then used from there:
```java
class PrintfImpls {
    static final FunctionDescriptor PRINTF_BASE_TYPE = FunctionDescriptor.of(JAVA_INT, ADDRESS);
    static final Linker LINKER = Linker.nativeLinker();
    static final Addressable PRINTF_ADDR = LINKER.defaultLookup().lookup("printf").orElseThrow();

    static MethodHandle specializedPrintf(MemoryLayout... varargLayouts) {
        FunctionDescriptor specialized = PRINTF_BASE_TYPE.asVariadic(varargLayouts);
        return LINKER.downcallHandle(PRINTF_ADDR, specialized);
    }

    public static final MethodHandle WithInt = specializedPrintf(JAVA_INT);
    public static final MethodHandle WithString = specializedPrintf(ADDRESS);
    public static final MethodHandle WithIntAndString = specializedPrintf(JAVA_INT, ADDRESS);
}
```

Note: The performance concern is that the JIT compiler (C2) will try to inspect and pull apart a method handle, to compile a call through a method handle like a call to any normal Java method.
But, it can only do this if the method handle is a constant (defined as a static final field).

## Conclusions

There is a strong correlation between the declaration of a native function and how it is invoked, so a function must be invoked exactly as it was declared, i.e., there's no flexibility.
This, unfortunately, impacts the implementation of the variadic function that requires having a function descriptor defined with the variadic argument layouts before the invocation.
Such an approach contradicts the nature of the variadic arguments as often the number of variadic arguments, their types, and the order is unpredictable.

So, Java developers would have to deal with this lack of flexibility and maintain the desired performance level
by making the runtime responsible for invocations, but not for constructing downcall method handles prior to the invocation of a native function.

Despite the not very pessimistic tone, there is another way to solve the problem of the variadic function by delegating the infrastructure code provisioning for native functions to Project Panama code tool –- jextract
whose goal is to generated Java source classes based on a C header file for the particular shared library. It will be covered in the next article

## Code listing

You may find sources for this chapter [here](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-2).
