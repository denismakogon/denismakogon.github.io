---
layout: post
title:  'Introduction to Project Panama. Part 2.1: Simply implementing variadic functions.'
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

This article explores a problem of native variadic functions in Java and a simplified implementation using Foreign Function and Memory API (Project Panama).

This article is divided into two pieces: one will cover a simplified solution, and another will dive into technical detail to explain the advanced solution.

## "C variadic functions in Java" problem

### Part 1 recap

The purpose of [Part 1]({{ '/openjdk/panama/2022/05/30/introduction-to-project-panama-part-1.html' | relative_url }})
was to make an introduction to the Foreign Function and Memory API. As a practical task, in Part 1, I have shown you how to implement a downcall to the C _printf_ function using corresponding API classes:
* **SymbolLookup** -- for searching native function address.
* **FunctionDescriptor** -- for declaring a Java-based function descriptor containing a return value and argument layouts that correspond to the original signature in C.
* **Linker** -- for creating a method handle for a native function from a descriptor and a native function address.
* **MemorySession** -- for native memory segment allocation and de-allocation.

### Revisiting C _printf_ Java implementation

[Part 1]({{ '/openjdk/panama/2022/05/31/introduction-to-project-panama-part-1.html' | relative_url }}) contains a simplified implementation of a downcall to the C _printf_ function in Java. That code did not implement most important feature -- C variadic arguments for _printf_ function.
The absence of variadic arguments made C _printf_ implementation look like _PrintStream::println_ rather than _PrintStream::printf_ in Java standard library.

There is no _println_ in C standard library, so the functionality of _println_, as we got used to, can be replaced with _printf_:
```cpp
// the same as 'println("Hello World")' if it existed
printf("Hello World!\n");
```

The most significant difference between _PrintStream::println_ and _PrintStream::printf_ is the support of varargs in the last one, i.e., the _PrintStream::println_ is a simplified version of _PrintStream::printf_.

The "Hello World" application in Part 1 did not implement the complete variadic function contract of C _printf_. This article will close the technical dept and explain how to develop a downcall to the C variadic functions using Java Foreign Function and Memory API.

### Variadic functions, variadic arguments

In C/C++, a variadic function, like _printf_, is a function type that takes a variable number of arguments indicated by the parameter of the form `...`
as a last one in the parameter list and must follow at least one named parameter:
```cpp
int     printf(const char * __restrict, ...);
```

At the moment of a call to the variadic function, it accepts variadic arguments. According to its definition, a length of variadic arguments could equal to zero but not greater than 127 entities per function call:
```cpp
printf("hello world");
printf(" I'm a=%s, I'm b=%s", a, b);
```

In C/C++, variadic arguments have no type defined at a function declaration. In Java, a situation is the opposite -- the type of varargs is required:
```java
Object someMethod(Object... varargs);
```
Often developers use broad varargs type definition using **Object** because most Java types inherit from it: any instance of an object which was type inherited from the **Object** will be accepted as a valid member of varargs at the runtime.

With all those facts described above, we need to answer how to implement C variadic arguments using Foreign Function and Memory API.

### Runtime representation of named and variadic arguments

In Part 1, the C _printf_ **FunctionDescriptor** had the following implementation:
```java
FunctionDescriptor function = FunctionDescriptor.of(
        JAVA_INT.withBitAlignment(32), ADDRESS.withBitAlignment(64)
);
```
As mentioned above, C _printf_ is a variadic function, but there is no sign of variadic arguments in the descriptor.

According to C _printf_ definition, it has both named (positional) args (`const char * __restricted`) and variadic arguments (`...`), but the Java entry point to the C _printf_ (**MethodHandle::invoke**) accepts varargs only (there is no distinction between named args and varargs):
```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
```

So, no matter what kind of combination of named and variadic arguments is provided (to **MethodHandle::invoke**) to call a native function, they must be supplied in two forms -- as array of type **Object[]**:
```java
var allArgs = new Object[] {};
// add a few args here
methodHandle.asSpreader(Object[].class, allArgs.length).invoke(allArgs); // explicit alternative to _MethodHandle::invokeWithArgument_
```
or passed as varargs where named args followed by the variadic preserving their order to comply with the **FunctionDescriptor** definition:
```java
methodHandle.invoke(one, two, three);
```

It is highly important to say that the **FunctionDescriptor** isn't just a representation of the C method signature (a return value, names arguments order, and variadic arguments) but also declares the Java method type of native function. Therefore, to perform a native call, the Java runtime must know what (a method handle) and how to call a native code (based on the method type).

#### Native function descriptor and a method type

The **FunctionDescriptor** is the key to a process of the native function invoke method type validation (or safe type converting) and the only entity the JVM relies on when attempting to match the **MethodHandle::invoke** method type and the descriptor method type as a reference value.

At the runtime, when the **MethodHandle::invoke** is called, the JVM attempts to perform safe type converting between
the method type (see [MethodType](https://download.java.net/java/early_access/jdk19/docs/api/java.base/java/lang/invoke/MethodType.html)) derived from a function descriptor
```java
//                named arg      ret-value
//                <ADDRESS>      <JAVA_INT>
MethodHandle  (  Addressable  )     int
```
and a method type created from the arguments passed to the **MethodHandle::invoke** (both named and variadic). The JVM will consider the method type mismatch as an exceptional situation.

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
System.out.println(toMethodType(descriptorWithNamedAndVariadicArg));
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

What we know so far is that the JVM will check if it can safely convert the invocation method type to a method type derived from the `function` descriptor:
```java
System.out.println(toMethodType(descriptorWithNamedAndVariadicArg));
```
```shell
(Addressable,Addressable,int)int
```

The attractiveness of variadic arguments is based on their variadic nature. There is no need to define another function with named arguments to call the  C _printf_ (or any other variadic function) with them as variadic.
However, in Java, a call to the variadic function makes developers prepare in advance -- it is necessary to create a descriptor for each possible variadic argument combination, as well as create a method handle from a newly created descriptor. So, the assumption that the following code should've worked is false:
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

These invocations will produce exceptions because a method type implicitly created from a set of variadic arguments
will not match the expected method type derived from the function descriptor used to create the `printfHandle` method handle.  So, the biggest problem for developers would be to keep a combination of variadic arguments compliant with a function descriptor. Every new named and variadic arguments combination will need a new function descriptor and eventually a new method handle based on it.

As you may see, there is a strong dependency between invocation parameters, a function descriptor, and a method handle. An invocation must always comply with the function descriptor, otherwise, it will fail.
Unfortunately, declaring in advance doesn't seem to be a complete solution.

## Choice between flexibility and simplicity

The arg types declaring in-advance solution will work in many cases, not all variadic functions accept a variety of types like _printf_.
However, the downside is the absence of flexibility. By sacrificing the flexibility we would have to turn variadic args into mostly named args, like in the case of _println_ and _printf_:
```java
int println(String str, String name int age) throws Throwable {
    FunctionDescriptor printfDescriptor =
            FunctionDescriptor.of(JAVA_INT, ADDRESS);
    MethodHandle printfHandle = linker.downcallHandle(addr, printfDescriptor);
    var formatter = SegmentAllocator.implicitAllocator().allocateUtf8String(str + "\n");
    var nameSegment = SegmentAllocator.implicitAllocator().allocateUtf8String(name);

    return (int) printfHandle.invoke(formatter, nameSegment, int);
}
```

So, it's either a choice between simplicity (creating a new function descriptor and a downcall handle for each variation of variadic argument types) or flexibility (complex, but fully automated solution).
In part 2, I will show how to implement the advanced solution.

## Code listing

You may find sources for this chapter [here](https://github.com/denismakogon/openjdk-project-samples/blob/master/Panama.md#openjdk-panama-part-2).
