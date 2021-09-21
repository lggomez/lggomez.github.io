{
    "title": "Memory: A Forgotten Topic",
    "description": "A post about one of the most overlooked topics by modern developers: memory",
    "summary": "A post about one of the most overlooked topics by modern developers: memory",
    "tags": ["C#"],
    "date": "2020-12-23T23:09:11-03:00",
    "lastmod": "2020-12-23T23:09:11-03:00",
    "categories": ["Programming", "Backend", "Theory"],
    "type": "article",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "memory-a-forgotten-topic",
    "url": ""
}

___***Note (December 2020): This is an article I wrote more than 3 years ago for my [employer at that time](https://engineering.hexacta.com/memory-a-forgotten-topic-355088e2f2c1), and now migrated to this space with some minor corrections. NET Core and CoreCLR Have grown exponentially in improvements and performance since the time of this post, but I also find interesting that most (if not all) of these concepts in software development still apply to this day, regardless of the platform***___

![1-1.jpeg](1-1.jpeg)

When we talk about memory in the programming scene we refer to the computer memory in which our program will reside and also, eventually, occupy by itself when it executes its operations
Anyone who had partaken into a programming course with an old agenda will know about this, because decades ago (when IT dinosaurs walked the earth) explicit and direct interaction with the memory was mandatory (think about C, C++, Pascal, ADA, and the list goes on…) and most of time it was even needed on a processor instruction level (Assembly).

¿What happened with this ancestral and challenging way of programming then? Not much. It’s quite true that with the advent of web and enterprise applications the explicit memory management started to be seen as an unnecessary workload, an art reserved to certain niches like mission and security critical applications, systems with high scalability and workload requirements, videogames, operative systems and drivers, among others
Nowadays, the majority of programmers that started their careers with more modern languages like Java, C# or Visual Basic, are alien to these concepts, even having cases which never had to work with pointers or memory addresses.

The purpose of this article is to do a high level recap of memory management in modern languages (using .NET as a base, although may apply to many others) and see which solutions related to memory management and allocation we can find.

#### Glossary 
  * **CLR**: Acronym for Common Language Runtime, is the .NET runtime, which is in charge of executing applications compiled in all of the .NET languages. Aside from the virtual machine and the Just-In-Time compiler it also has additional responsibilities like memory management, security, and so on
  * **BCL**: Acronym for Base Class Library, is the core library of the .NET framework. Besides operating directly with the CLR it exposes the primitive data types and essential functionality to build and run an application. Also known as mscorlib

## Managed vs unmanaged memory 
Languages and development platforms of today rely in several techniques to manage the memory employed by its aplications in collaboration with the operative system to provide a great amount of abstraction, but it also lies along with the counterparts that don’t employ such benefits and do a manual memory management. 

As a consequence of this differentiation in practice, two concepts come up:
  * **Managed memory**: This type of memory is the one used by our traditional objects and classes, that don’t use resources or third party libraries. It’s not the only example although it’s the biggest one. In Java, these are known as POJO (Plain Old Java Object) and it’s analogous in .NET is POCO (Plain Old CLR Object). This memory is managed automatically by a subsystem specially designed for that purpose, the garbage collector (which we’ll see in the next section). We call managed languages to those that apply several abstraction measures to prevent or forbid the direct memory usage on behalf of the programmer
  * **Unmanaged or native memory**: This group encloses the rest of the applications that do their own memory management, usually made on a lower level language that doesn’t make use of a garbage collector like C or C++, even though there are hybrid alternatives like Rust and D that give the option of having a garbage collector at runtime. Resources deserve a special mention, they are another group that encompass hardware related code and pointers, external systems and the operative system, with concepts like handles, connections, I/O and OS primitives (synchronization, security, etcetera)

Following the previous reasoning, the availability of unmanaged elements in managed scenarios is due to the fact that some managed objects act as wrappers of resources to employ, and handle accordingly the memory usage and disposal of these. Examples can be found in the APIs `System.IO` or `System.Runtime.InteropServices`

The communication process between libraries, in the context between managed and unmanaged is called interop (as in interoperability), and the process of transition between native memory to managed objects is called marshalling. We can think of it as a serialization analogy but between native memory and our application objects. In .NET, the interop features are provided by the BCL, via the `InteropServices` namespace along with the `static extern` C# keywords, to define the signatures of the external methods to call. It’s also known as P/Invoke

## The garbage collector: an unexpected ally 
Now that we had a humble introduction to memory management, it’s the turn of one of its main protagonists: the garbage collection (we will refer to it as GC). It consists of a subsystem that usually does the cleanup of memory that is no longer used by our application (and in some cases, like .NET, also handles the memory allocation). 

The design and implementation of garbage collectors is an extremely complex topic that span entire books, and the implementation details of the .NET GC alone deserves a separate article, so we will just summarize the main features, objectives and responsibilities that takes on its own:
  * Allocation and defragmentation of memory, being the main component of the CLR regarding memory management along with the execution engine
  * It comes in blocking (single thread) and parallel (multi thread­) versions. Both can have very different effects in the performance of the same program
  * It’s of the mark & sweep and generational types, which means that instead of following all of the objects’ life cycles it only marks them in order to collect them once they are dereferenced, and are divided into generations according to their age. Collections are executed one generation at a time
  
The main objectives of a GC are:
  * **Efficiency**: Do quick and infrequent collections
  * **Efficacy**: Collect most (if not all) of the dereferenced memory
  * **Correctness and consistency**: False positives are unacceptable. Called GC holes, are instances of objects that are wrongly marked as dereferenced. It also has to mark, eventually, all objects that are dereferenced

To achieve these objectives, it relies on a lot of heuristics and optimizations along with the CLR.

It’s also important to know that it’s non deterministic. This means that the developer cannot control the behavior of the GC at will, it can only receive hints to its heuristics.

Managed languages contain a given implementation of a GC, but there cases like Java that have several implementations, some of them commercial and developed by third party vendors. For more information about the .NET garbage collector I recommend Maoni Stephens’ blog, she is the development lead of the .NET GCs.

## Memory leaks and other mythical beasts 
As we saw previously, the GC does a great job when it comes to manage the memory that we, the developers, use, although it can’t manage all scenarios, so it depends on our help and good practices to keep doing its job properly. This is especially true in large applications that make extensive use of resources and have large object trees.

The first issues that arise due to a poor memory management are memory leaks. These situations occur when there are idle objects in memory that are no longer needed, but are still referenced by other objects, so they can’t be claimed by the GC. This also happens when we do an inappropriate cleanup of unmanaged memory, because it’s out of bounds for the GC (unless it’s part of a managed wrapped being collected).

A classic example of a memory leak caused by the programmer’s negligence is the following:

```csharp
static event EventHandler Leaker;

for(int i = 0 ; i < 100000 ; i++) {
  Leaker += delegate {};
}
```

In this scenario, a massive event handler registration is being made on a static field, which implies that these objects will never be claimed because these never get out of a scope and get dereferenced. The events hold the subscribers’ references, and if not controlled, they will reside in memory forever.

Another variant is what happens when object life cycles are extended (even if we know that they are going to be dereferenced and collected). This can induce temporary memory leaks due to object recollection retardation, and in some cases it can cause exceptions due to thread safety or resource consumption issues.

Finally, we have the most unusual but dangerous variant, the handle leaks. Leaking handles is more risky than leaking memory due to a simple reason: the CLR is extremely good at managing memory, but handles are a native OS resource, and the CLR has no say over them. Handle starvation and misuse can cause widespread problems like device access issues, OS instability, memory corruption and crashes.


## Memory leaks: a personal experience 
A personal horror story in .NET goes back to the abuse of coroutines (yield return) and extended life cycles in parallel code. The original code I had to maintain did the following:
  * A ForAll expression was run over a collection of objects that retrieved data from a database
  * These objects consumed a singleton DataReader via a coroutine, using connected ADO
  * The code was parallel, so multiple calls were being made towards the DataReader at the same time
  * To prevent invalid access exceptions (since DataReader is not safe) the database call section was blocking, which “assured” that the code was thread safe

Trusting the original code was a mistake, since the autor and I forgot a detail that seemed minor but took me hours and hours of debugging: the coroutines capture the objects being exposed via the IEnumerator returned by it, which extends the life cycle of the object. This delayed the DataReader object and resource finalization, thus resulting in unpredictable DataReader access exceptions

The moral of the story is: trust no one, not even your language of choice’s syntactic sugar

## Managed memory in .NET 
In .NET, all objects are allocated in memory (be it stack, heap or the large object heap — LOH) in 3 ways: from the JIT compiler, from the compiled code o directly to the LOH (for more info check this). Direct allocation of data into register is done later, during the parameter passing and optimizations done during JIT compilation.

Given that, we can infer that all objects will be eventually collected by the GC, assuming that the developer was responsible and didn’t cause any memory leaks. Later on, the objects are directly collected and the memory freed, although there are exceptions. In C# we have finalizers, which is an analog of C++ class destructors. Instances of class that implement them will be taken by the GC upon collection and put into a finalization queue, separated from traditional objects, that will dispatch objects as soon as their internal references are collected, and after that, it will trigger the finalizer of said instance. This stage is non deterministic and we cannot control the exact moment nor order of execution.

We have 2 cases that enter in this category:
  * `IDisposable` objects: The IDisposable interface is special because, depending on the implementation, it will have different behaviors at GC time. A complete implementation of the pattern with unmanaged resources that use finalizers will incur in the behavior we explained above
  * `CriticalFinalizerAttribute`: This attribute signals the CLR to put the instance in the last place of the finalization queue, and ensures that the finalizer will be run, regardless of the state of the execution engine. It’s used in low level classes at the BCL level like SafeHandle, a managed wrapper for handles that will dispose the handle even if the program finishes its execution in an anomalous fashion

## Unmanaged and native memory in C# 
The presence of unmanaged memory in .NET applications is reduced to these scenarios:
  * **Interop**: This is the most usual use case, be it for communications with third party libraries or WinAPI. Concepts like marshalling, calling conventions and OS handles are important here, and all classes that implement these methods and/or have access to handles should be recycled correctly via the `IDisposable` pattern, and consumed via the using keyword
  * **Resources**: They were mentioned in the point above but it’s worth noting that resource usage is a source of unmanaged memory. I/O, direct memory, GPU or connections to servers, these generate memory which is not handled by the CLR, but counts as memory in terms of usage by our program. Memory leaks of this type may be harder to detect than their managed counterparts
  * **Unsafe code**: A relatively unknown feature of C# is the possibility to define methods and code areas with the unsafe keyword, which gives the developer the capabilities of pointer operations and object pinning, so it won’t be moved by the GC upon memory compaction, thus allowing to operate with memory addresses and offsets. It’s rarely used, but valuable in extreme microoptimization scenarios
  * **Unmanaged APIs**: There are methods that allow to allocate unmaged memory in an explicit way, and return a handle (IntPtr class in the BCL) to manipulate said memory. If misused, this is a really easy way to leak memory and induce exceptions of type `AccessViolationException`, which are not manageable as regular exceptions

The following example shows how to save and read number in unmanaged memory via the Marshall API

```csharp
namespace MemDemo
{
    using System;
    using System.Runtime.InteropServices;
    class AllocDemo
    {
        static void Main()
        {
            //Calculate the int size since it's architecture dependent
            const int int32Size = sizeof(int);
            //Allocate the memory for 10 numbers
            IntPtr hglobal = Marshal.AllocHGlobal(int32Size * 10);
            //Write the numbers into memory using the count as offset
            for (var i = 0; i < 10; i++)
            {
                Marshal.WriteInt32(hglobal, int32Size * i, i);
            }
            Console.WriteLine($"{int32Size} bytes of memory allocated and initialized.");
            //Read the numbers from memory using the count as offset
            string values = "{";
            for (var i = 0; i < 10; i++)
            {
                values += Marshal.ReadInt32(hglobal, int32Size * i) + " ";
            }
            values += "}";
            Console.WriteLine($"{int32Size} bytes of memory read. Values are {values}");
            //Free the used memory, otherwise we'd causing a memory leak
            Marshal.FreeHGlobal(hglobal);
            Console.WriteLine("Memory deallocated successfully");
            Console.ReadLine();
        }
    }
}
```

And in the following example we’ll do a special hello world, using the kernel native method to output the message. This will be printed to the application debugger output (in Visual Studio it can be seen in the Output window). This operates entirely out of the heap space, as it interoperates with the Windows Kernel API:

```csharp
namespace MemDemo
{
    using System;
    using System.Runtime.InteropServices;
    class WinapiDemo
    {
        [DllImport("kernel32.dll")]
        static extern void OutputDebugString(string lpOutputString);
        static void Main()
        {
            OutputDebugString("Hello world!");
            Console.ReadLine();
        }
    }
}
```
