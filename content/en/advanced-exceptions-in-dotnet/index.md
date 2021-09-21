{
    "title": "Advanced exceptions in .NET",
    "description": "A tour on advanced exceptions in .NET (circa 2016)",
    "summary": "A tour on advanced exceptions in .NET (circa 2016)",
    "tags": ["C#", "Generics"],
    "date": "2020-12-26T18:52:29-03:00",
    "categories": ["Programming", ".NET", "Advanced", "Tutorial"],
    "type": "publication",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "advanced-exceptions-in-dotnet",
    "url": ""
}

___***Note (December 2020): This is an article I wrote almost 4 years ago for my [employer at that time](https://engineering.hexacta.com/advanced-exceptions-in-net-f68debf990c4), and now migrated to this space with some minor corrections.***___

The goal of this article is to give you a **tour around advanced exceptions in .NET, in order to learn about their pitfalls and best practices** when developing applications.

![2-1.jpeg](2-1.jpeg)

Exception handling is a vital part of an application, no exceptions (pun intended).

The efficacy of this handling depends mostly on the language and platform of choice, so it’s important to know in detail their correct usage and behavior in order to spare our users and fellow developers from suffering when diagnosing issues in our code.

In this article we’re going to take a look at what C# and .NET do in regards of error handling.

## Glossary 
* **CLR**: Acronym for Common Language Runtime, is the .NET runtime, which is in charge of executing applications compiled in all of the .NET languages. Aside from the virtual machine and the Just-In-Time compiler it also has additional responsibilities like memory management, security, and so on.
* **BCL**: Acronym for Base Class Library, is the core library of the .NET framework. Besides operating directly with the CLR it exposes the primitive data types and essential functionality to build and run an application. Also known as mscorlib.
* **FCL**: Acronym for Framework Class Library, is what most of us know as “the framework” in .NET. Using the BCL as building blocks it exposes a great deal of namespaces with various features such as System.IO, System.Security, System.Text, among others.
* **TPL**: Acronym for Task Parallel Library, is the library that contains the functionality provided by the async keywords and API’s. Released with .NET version 4.5, along with C# 5.
* **SEH**: Acronym for Structured Exception Handling, is the native exception subsystem of Windows, it handles software and hardware exceptions on the operating system level
* **MDA**: Acronym for Managed Debugging Assistants. These are special debug extensions that provide information related to the CLR execution state to the Visual Studio debugger, which is exposed by internal helpers and assets. See the [MSDN](https://msdn.microsoft.com/en-us/library/d21c150d.aspx) page for more details

## Exceptions Revisited 
**In .NET, and particularly in C#, exceptions are handled using try, catch and finally blocks**. First of all, the try block will contain the code which is expected to throw an exception, and in second place we will have a catch block, that will specify an exception type and a code block that will be executed in the event that an exception matching the specified type is thrown inside the try block:

```csharp
Random rnd = new Random();
try {
	Console.WriteLine(1 / rnd.Next(-100, 101));
}
catch (DivideByZeroException ex) {
	Debug.WriteLine(“A division by zero attempt has occurred”);
}
```

It’s worth noting that an Exception type catch will handle all possible exceptions (this is not entirely true but we will stick with this assumption for now):

```csharp
try {
	//...
}
catch (Exception ex) {
	//General exception handler
}
```

Another way of writing the general exception handler is the following:

```csharp
catch {
	//...
}
```

A try block can have multiple catch blocks with different exception types associated, and those will be evaluated according to class hierarchy in a descending order:

```csharp
catch (DivideByZeroException ex) {
	 //Code that handles division by zero exceptions
}
catch (Exception ex) {
	 //Code that handles any exception that may occur
}
```

**NOTE**: It’s important to **keep in mind the evaluation order of exception types**, since if our first catch handles an Exception type, all of the exception handlers below will be ignored. This is a common error that might be difficult to detect later in complex applications.

And finally, we have the finally block (apologies for the redundancy), which is going to execute its contained code always, whether an exception has been thrown or not. So, a complete exception handler has the following form:

```csharp
try {
    //...
}
catch {
    //...
}
finally {
    //...
}
```

It is worth noting (again) that the code inside a finally block isn’t included in the exception handling, so any exception thrown here won’t be caught, and it will bubble up the call stack until it’s managed.

As a last detail we can add that the **throw keyword has a special meaning in the exception handling context**, because it can also allow us to specify how to propagate the exception caught. Based on the previous example:

```csharp
try {
	Console.WriteLine(1 / rnd.Next(-100, 101));
}
```

We can propagate the exception in several ways inside the catch block:

```csharp
catch (DivideByZeroException ex) {
	//1 – Continue with the exception while preserving its stack trace, also called rethrow
	throw;
	//2 – Throw the same type of exception but eliminating its original stack trace
	throw ex;
	
	//3 – Throw a new exception, eliminating the original stack trace
	throw new InvalidOperationException();
	//4 - Throw a new exception, passing the current one as its InnerException
	throw new InvalidOperationException(ex);
}
```

**A known limitation of async in C# is that the await keyword cannot be used inside catch and finally blocks. This was resolved in C# 6**, and, since we’re talking about C# 6 we are going to mention a new feature added to the catch blocks, the exception filters. This new syntactic sugar lets us capture or propagate an exception according to the boolean value of an expression:

```csharp
try
{…}
catch (MyException e) when (filter(e))
{…}
```

One of the major advantages is that we can manage the exception flow without altering the stack trace. It can be also used as an interceptor, to add side effects like logging:

```csharp
private static bool Log(Exception e) { 
  /* Exception logging */; 
  return false; //This preserves the call stack
}
try 
{...} 
catch (Exception e) when (Log(e)) 
{...}
```

We still have to keep in mind the exception order we saw previously, to avoid issues like the following:

```csharp
catch (MyException ex)
{
    // Will catch all instances of MyException
}
catch (MyException ex) when (filtro1(ex) && filtro2(ex))
{
    // Unreachable
}
catch (MyException ex) when (filtro1(ex))
{
    // Unreachable
}
```

## Exceptions in .NET 
The table below shows a list with all base exceptions from the BCL (source: MSDN)

> **Exception — Base Type: Object**
  
  Base class for all exceptions.

> **SystemException — Base Type: Exception**
  
  Base class for all runtime-generated errors.

> **IndexOutOfRangeException — Base Type: SystemException**
  
  Thrown by the runtime only when an array is indexed improperly.
  
> **NullReferenceException — Base Type: SystemException**
  
  Thrown by the runtime only when a null object is referenced.
  
> **AccessViolationException — Base Type: SystemException**
  
  Thrown by the runtime only when invalid memory is accessed.
  
 > **InvalidOperationException — Base Type: SystemException**
  
  Thrown by methods when in an invalid state.
  
 > **ArgumentException — Base Type: SystemException**
  
  Base class for all argument exceptions.
  
 > **ArgumentNullException — Base Type: ArgumentException**
  
  Thrown by methods that do not allow an argument to be null.
  
> **ArgumentOutOfRangeException — Base Type: ArgumentException**
  
  Thrown by methods that verify that arguments are in a given range.
  
> **ExternalException — Base Type: SystemException**
  
  Base class for exceptions that occur or are targeted at environments outside the runtime.
  
> **COMException — Base Type: ExternalException**
  
  Exception encapsulating COM HRESULT information.
  
> **SEHException — Base Type: ExternalException**
  
  Exception encapsulating Win32 structured exception handling information.
  We can also include the type [AggregateException](https://msdn.microsoft.com/en-us/library/system.aggregateexception(v=vs.110).aspx) that results from the execution of tasks and async code (TPL and Parallel LINQ). It’s a special exception type because it consolidates multiple exceptions in a single object, composing a tree of exceptions. For more information on usage techniques and use common use cases see the [tasks](https://msdn.microsoft.com/en-us/library/dd537614(v=vs.110).aspx) and [PLINQ](https://msdn.microsoft.com/en-us/library/dd460712(v=vs.110).aspx) articles on MSDN

In C# there are several keywords that include exception handling in their generated code:
  * **Using**: Generates a try/finally, in which the Dispose method of the containing object is executed.
  * **Async/await**: Async code is compiled to a state machine that manages method invocation transitions and delegation, and, among other things, it generates catch blocks that aggregate all exceptions thrown.
  * **Yield**: In a similar fashion to async, coroutines (implemented in C# via the yield return statements) also generate a state machine with its respective exception handling code.
  
## Exception dispatch
One of the new features available since .NET 4.5, as part of the TPL release is the exception dispatch, facilitated by the class [ExceptionDispatchInfo](https://msdn.microsoft.com/en-us/library/system.runtime.exceptionservices.exceptiondispatchinfo(v=vs.110).aspx) of the namespace `System.Runtime.ExceptionServices`.

With this class we can preserve an exception and delegate it to another instance, as long as we stay in the same AppDomain. Later on we can inspect the exception and throw it again (rethrow):

```csharp
ExceptionDispatchInfo exInfo = null;
try
{
 //...
}
catch (Exception ex)
{
  exInfo = ExceptionDispatchInfo.Capture(ex);
}
//...
if (exInfo != null)
{
  exInfo.Throw();
}
```

## Exceptional exceptions: Part I — CSE 
Up until now we had assumed that no matter what happens, all exceptions will be unconditionally caught by their associated catch blocks, and later on the finally block will be executed. Is this still true?
The answer is no, at least not always. By design, the CLR has some special exceptions types that cannot be captured, or at least they cannot be captured without forcing it via configuration.
The first kind are the Corrupted State Exceptions (CSE). They consist of a **group of 8 native exceptions from Win32/SEH** that, due to their nature, undergo the assumption that they’re not manageable since they imply that the program is in an invalid, inconsistent and unrecoverable state.
**The native exceptions, along with their CLR counterparts, are the following:**
	* EXCEPTION_ACCESS_VIOLATION — System.AccessViolationException
	* EXCEPTION_STACK_OVERFLOW — System.StackOverflowException
	* EXCEPTION_ILLEGAL_INSTRUCTION — SEHException
	* EXCEPTION_IN_PAGE_ERROR — SEHException
	* EXCEPTION_INVALID_DISPOSITION — SEHException
	* EXCEPTION_NONCONTINUABLE_EXCEPTION — SEHException
	* EXCEPTION_PRIV_INSTRUCTION — SEHException
	* STATUS_UNWIND_CONSOLIDATE — SEHException

At this instance, the exceptions are shown as “Unhandled exception” and are not observable from the local debug in Visual Studio. The only way to diagnose these in more detail is to use with its [SOS](https://msdn.microsoft.com/en-us/library/bb190764(v=vs.110).aspx) extension and analyzing crash dumps, among others.

The reason behind them being ignored by managed code is that **they are incoming exceptions from native code, which escape the application’s responsibility or, alternatively, we are dealing with a CLR bug**, situation that is very unlikely but not impossible. It can also be a result of bugs in our code if we are using the unsafe feature of C# to write unmanaged code (with direct access to memory).

If we still wish to handle these exceptions, the CLR gives us the choice to catch some (not all) of them via the attribute `HandleProcessCorruptedStateExceptions`. This requires the method having an access level of `SecurityCritical`:

```csharp
[HandleProcessCorruptedStateExceptions]
[SecurityCritical]
public void MyMethod()
{
	try
	{
		Marshall.StructureToPtr(1000, new IntPtr(1000), true);
	}
	catch {…}
}
```

This can also be achived on a whole process by specifing the `legacyCorruptedStateExceptionPolicy` attribute as true, in our application configuration.

**NOTE**: Due to compatibility reasons, The CSE’s will be removed in the upcoming versions of .NET Core. These will remain untouched in the desktop version, since it’s the original Windows version.
There are 2 remaining exceptions types that cannot be handled and indicate a premature and unrecoverable process termination: the `StackOverflowException`s (if originated within the CLR), and the CLR exceptions, which we’ll see next.

## Exceptional exceptions: Part II — Exceptions in the CLR 
Diving deeper into the origin of exceptions, we stumble upon the CLR. The runtime is mostly build in C++, and has the task of dominating a complex scenario which is the handling of multiple exception types:
  * **Native Windows exceptions (SEH)**: The CLR unifies the handling of these exceptions with macros that simulate the VC++ compiler intrinsics (__try, __catch, etcecera). The SEH model is extremely intricate, making the unwind (we’ll see about that in the next section) a costly process in regards to stack frames and binary compatibility in comparison to linux, for example
  * **C++ exceptions**: These exceptions can be thrown by the CLR code itself
  * **CLR exceptions**: These are the exceptions delegated and exposed by the virtual machine to the application during execution

CoreCLR, the multiplatform and open source version of the CLR, has the even harder task of integrating compatibility with multiple platforms and architectures (Linux and Mac x64, among others in progress like x86 and ARM32/ARM64). **To achieve the level of decoupling required, it relies on a layer called PAL (Platform Adaptation Layer).**

In the event of a critical error inside the CLR it will throw an ExecutionEngineException (deprecated) and FatalExecutionEngineError (MDA). These errors commonly indicate heap corruptions in managed code.

## Performance Implications 
To finish the exceptions tour, we’ll move on to a short but obligatory talk about the performance implications they have.
Exception handling is very expensive compared to the execution of normal code. An exception incurs in the following actions and side effects:
* Cache errors and page faults in memory due to the execution flow (and context, consequently) changes.
* Unwind: this is the process in which the context belonging to the exception causing code in the try block is “cleaned”. This involves traversing all of the previous stack frames and the deallocation/disposing of the affected objects. All of this, additionally, may bring forward the next memory compaction by the garbage collector.
* Additional allocations for the diagnostic objects (StackTrace)
* “Cold” code accesses, which demand additional time for Just In Time compilation (JIT).
In normal situations there won’t be a noticeable performance impact, but the abuse of exceptions in situations with tight performance or scalability requirements can be troublesome.

To finish the article we hand out some tips that can be useful when designing and implementing efficient solutions:
* Avoid using exceptions as a control flow in our application. Use error codes or, even better, contemplate the errors and warning as part of your class design.
* Be measured with the exception throwing and catching: in web scenarios, for example, the exception handling can be centralized per request, or, in layered architectures, they can be handled per invocation across layers (dependency injection frameworks offer several tools for that end).
* Avoid having try blocks with excessive amounts of code, especially if the throwing code isn’t immediately related or dependent on it. This is because large code bodies (in both methods and blocks) can preemptively disable all runtime optimizations made by the Just In Time compiler (JIT).
* Currently (as of .NET Core 1.1) methods with try/catch blocks are not inlined (not even with MethodImplOptions.AggressiveInlining). Keep this in mind in order to prevent surprises
