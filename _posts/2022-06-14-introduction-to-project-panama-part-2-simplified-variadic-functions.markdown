---
layout: post
title:  'Introduction to Project Panama. Part 2: Simply implementing variadic functions.'
date:   2022-06-6
categories: openjdk panama
tags: ["openjdk", "panama"]
image_src_url: 'https://unsplash.com/photos/Wiwqd_8Rds8/download?ixid=MnwxMjA3fDB8MXxzZWFyY2h8OXx8cGFuYW1hfGVufDB8fHx8MTY1NDMyNzMwMg&force=true&w=800'
excerpt: 'This article explores problems of variadic functions and possible implementations of variadic functions using Foreign Function and Memory API.'
---

![Panama]({{ '../images/openjdk-panama/luis-gonzalez-Wiwqd_8Rds8-unsplash.jpg' | relative_url }})
Photo by [Luis Gonzalez](https://unsplash.com/@luchox23?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on [Unsplash](https://unsplash.com/s/photos/panama?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


## Intro

This article explores Java native variadic functions and implementation using Foreign Function and Memory API (Project Panama).

There are two solutions to the problem of the C variadic functions in Java. This article will focus on a simplified implementation. The next one will cover an advanced solution.

## C Variadic Functions in Java

### Part 1 recap

The purpose of [Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }})
was to make an introduction to the Foreign Function and Memory API. As a practical task, Part 1 shown you how to implement a downcall to the C _printf_ function using corresponding API classes:
* **SymbolLookup** -- to lookup a native function memory address.
* **FunctionDescriptor** -- to declare a Java-based function descriptor containing a return value and argument layouts that correspond to the original signature in C.
* **Linker** -- to create from a function descriptor and its native function address a (Java) method handle wrapping the native function
* **MemorySession** -- for native memory segment allocation and de-allocation.

### Revisiting C _printf_ Java implementation

[Part 1]({{ '/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html' | relative_url }}) contains a simplified implementation of a downcall to the C _printf_ function in Java.
That code did not implement a key (or important) feature -- C variadic arguments for _printf_ function.
The absence of variadic arguments made C _printf_ implementation look like _PrintStream::println_ rather than _PrintStream::printf_ in Java standard library.

The most significant difference between _PrintStream::println_ and _PrintStream::printf_ is the support of varargs in the last one, i.e., the _PrintStream::println_ is a simplified version of _PrintStream::printf_.

The "Hello World" application in Part 1 For the sake of simplicity part 1 did not implement the variadic contract of C _printf_. 
This article contains technical detail of the process and explains the development of a downcall to the C variadic function using Java Foreign Function and Memory API.

### Variadic functions, variadic arguments

In C/C++, a variadic function, like _printf_, is a function type that takes a variable number of arguments indicated by the parameter of the form `...`
as in the latter in the parameter list and must follow at least one named parameter:
```cpp
int     printf(const char * __restrict, ...);
```

At the moment of a call to the variadic function, it accepts no more than 127 variadic arguments per function call:
```cpp
printf("hello world");
printf(" I'm a=%s, I'm b=%s", a, b);
```

In C/C++, variadic arguments have no type defined at a function declaration. In Java, it is the opposite -- the type of varargs is required:
```java
Object someMethod(Object... varargs);
```
Java developers often use broad varargs type definition using **java.lang.Object** because all Java types inherit from it.

With all those facts described above, we need to answer how to implement C variadic arguments using Foreign Function and Memory API.

### Runtime representation of named and variadic arguments

In Part 1, the C _printf_ **FunctionDescriptor** had the following implementation:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
The C _printf_ is a variadic function, but there is no sign of variadic arguments in the descriptor.

According to C _printf_ definition, it has both named (positional) args (`const char * __restricted`) and variadic arguments (`...`) while the Java entry point to the C _printf_ (**MethodHandle::invoke**) accepts varargs only (there is no distinction between named args and varargs):
```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
```

So, no matter what kind of combination of named and variadic arguments is provided (to **MethodHandle::invoke**) to call a native function, they must be supplied in two forms -- as array of type **java.lang.Object[]**:
```java
var allArgs = new Object[] {};
// add a few args here
methodHandle.asSpreader(Object[].class, allArgs.length).invoke(allArgs); // explicit alternative to _MethodHandle::invokeWithArgument_
```
or passed as varargs where named args followed by the variadic preserving their order to comply with the **FunctionDescriptor** definition:
```java
methodHandle.invoke(one, two, three);
```

The **FunctionDescriptor** isn't just a representation of the C method signature (a return value, names arguments order, and variadic arguments) but also declares the Java method type of native function. 
Therefore, to perform a native call, the Java runtime must know the method handle and how to call the native code based on the method type.

#### Native function descriptor and a method type

The **FunctionDescriptor** is the key to Invoking method type validation for a native function (safe type converting) and the only entity the JVM relies on when attempting to match the **MethodHandle::invoke** method type to the descriptor method type as a reference value.

At runtime, MethodHanlde::Invoke is called, and the JVM attempts safe type converting between the [MethodType](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodType.html) derived from a function descriptor
```java
//                named arg      ret-value
//                <ADDRESS>      <JAVA_INT>
MethodHandle  (  Addressable  )     int
```
and a method type created from the arguments passed to the **MethodHandle::invoke** (both named and variadic). The JVM will recognize type mismatch as an exceptional situation.

In the case of the C _printf_, the JVM will check if it can safely convert a combination of named and variadic arguments submitted to the **MethodHandle::invoke** to a method type derived from the Java implementation of the C _printf_ function descriptor.

In Part 1, the **MethodHandle::invoke** was called with a Java **String** object enclosed into a memory segment (**MemorySegment**):
```java
MemorySegment cString = memorySession.allocateUtf8String(str + "\n");
int res = (int) printfMethodHandle.invoke(cString);
```
At invocation, the JVM will create a method type from arguments (`MethodHandle(MemorySegment)int`), then attempt to convert that it to a method type derived from the function descriptor:
```java
//    <descriptor method type>       <invoke method type>
( MethodHandle(Addressable)int ) MethodHandle(MemorySegment)int
```

Note: Such type converting procedure would be successful because the **MemorySegment** class implements the [Addressable interface](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/Addressable.html).

The key takeaway is that the JVM will use the **FunctionDescriptor** return value and argument layouts to create a method type.
So the argument types, order, and quantity validation will be enforced by the Java runtime at the call of a native function because with a return value type and the arguments form a method type:
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


## Implementing C variadic functions using Foreign Function and Memory API

A new API offers a method for the [explicit variadic arguments definition](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/foreign/FunctionDescriptor.html#asVariadic(java.lang.foreign.MemoryLayout...)):
```java
public FunctionDescriptor asVariadic(MemoryLayout... variadicLayouts)
```

The argument layouts provided to **FunctionDescriptor::asVariadic** will also become a part of a method type following the named arg types:
```java
var descriptorWithNamedAndVariadicArg = descriptorWithNamedArg
        .asVariadic(ADDRESS, JAVA_INT);
System.out.println(Linker.downcallType(descriptorWithNamedAndVariadicArg));
```
```shell
(Addressable,Addressable,int)int
```

Let's say we want to implement the following C _printf_ downcall:
```cpp
printf("My name is %s, age %d\n",  "Denis", 31)
```

The C _printf_ will have to accept _char * p_ and _int_ as variadic arguments, so the function descriptor must declare them explicitly in the corresponding order:
```java
FunctionDescriptor descriptorWithNamedAndVariadicArg = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
).asVariadic(ADDRESS.withBitAlignment(64), JAVA_INT.withBitAlignment(32));
```

Compared to the C _printf_ definition, the `descriptorWithNamedAndVariadicArg` descriptor holds more details regarding variadic arguments (types, order, and quantity). A call to a function defined by such description will have the following look:
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

The attractiveness of variadic arguments is based on their variadic nature.
However, a call to the variadic function makes Java developers prepare in advance -- it is necessary to create a descriptor for each possible variadic argument 
combination and a method handle from a newly created descriptor. So, the following code will fail:
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

These invocations will produce exceptions because of the actual method type and the expected type mismatch. 
So, the biggest problem for developers would be to keep a combination of variadic arguments compliant with a function descriptor. Every new named and variadic arguments combination will need a new function descriptor and eventually a new method handle based on it.

There is a strong dependency between invocation parameters, function descriptors, and method handles. An invocation **must** always comply with the function descriptor. If not -- it will lead to the exception.
Unfortunately, declaring in advance does not ensure that the code will work.

## Concerns

### Impact on the flexibility

The arg types declaring an in-advance solution will work in many cases. Not all variadic functions are like the C _printf_ that accept variadic arguments of different types.
However, the downside is the absence of flexibility. By sacrificing the variadic arguments flexibility, they turn into mostly named args, like in the case of _println_ and _printf_:
```java
int println(String str, String name, int age) throws Throwable {
    var allocator = SegmentAllocator.implicitAllocator();
    FunctionDescriptor printfDescriptor =
            FunctionDescriptor.of(JAVA_INT, ADDRESS).asVariadic(ADDRESS, JAVA_INT);
    MethodHandle printfHandle = linker.downcallHandle(addr, printfDescriptor);
    var formatter = allocator.allocateUtf8String(str + "\n");
    var nameSegment = allocator.allocateUtf8String(name);

    return (int) printfHandle.invoke(formatter, nameSegment, age);
}
```

### Impact on the performance

The biggest concern is the performance impact as there are too many moving pieces, i.e., on every call the runtime will create a new instance of a method handle.
The advantage that the C compiler has is that it can see the argument types that are passed to a particular variadic call, and compile accordingly.
In the Java case, the **Linker** takes on the role of the compiler, and it has to be told in terms of the layout/function descriptor API which argument types are used.

For performance reasons, a method handle for each variadic argument combination should be stored inside a static final field end then used from there:
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
The resulting method handles should be stored inside a static final field end and then used from there:
```java
PrintfImpls.WithIntAndString.invoke(formatter, 42, stringMemorySegment);
```

## Conclusions

There is a strong dependency between the declaration of a native function and how it is invoked, so a function must be invoked exactly as it was declared, i.e., there's no flexibility.
Unfortunately, it impacts the implementation of the variadic function requiring to have a function descriptor defined with the variadic argument layouts before the invocation.
An approach like this contradicts the nature of the variadic arguments because often the number of variadic arguments, their types, and the order are unpredictable.

So, Java developers would have to deal with the impacted flexibility as well as maintain desired performance level
by making the runtime responsible for invocations, but not for creating downcall handles before the invocation.

Despite the not very pessimistic tone, there is yet another way to solve the problem of the variadic function by delegating
the infrastructure code provisioning for native functions to Project Panama code generating tool -- jextract.

## Code listing

You may find sources for this chapter [here](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-2).
