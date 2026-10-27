# .NET Runtime Notes: CLR, IL, JIT, and Azure

_Last updated: 2026-06-30_

## Core Idea
In .NET, source code is compiled into **IL/CIL + metadata** by the **compiler (Roslyn)**. At runtime, the **CLR** loads that code, provides runtime services, and uses **JIT** to turn IL into native machine code.

## 1) Compiler (Roslyn)
The **C# compiler** is a **separate entity** from the CLR — it is a **build-time tool**, not a runtime component.

### Key facts
- Named **Roslyn** — open-source, maintained by Microsoft
- Delivered as part of the **.NET SDK** (which you install on your dev machine or CI server)
- It **is** an executable/app — unlike CLR, it runs as a standalone build tool
- Invoked via `dotnet build` or your IDE
- Produces: **IL/CIL + metadata** (stored in `.dll` / `.exe` assemblies)

### Cross-platform and cross-compilation
- The Roslyn source code is **the same across platforms**
- OS-specific binaries exist (Windows, Linux, macOS versions of the compiler)
- **Cross-compilation is possible**: compile *on* macOS and produce binaries *for* Windows:
  ```
  dotnet publish -c Release -r win-x64
  dotnet publish -c Release -r linux-x64
  ```
- Without `-r`, it targets the current machine's OS by default

### What the compiler is not
- not the CLR
- not JIT
- not a runtime component — it does not run alongside your app

## 2) CLR
**CLR (Common Language Runtime)** is the **runtime execution engine** for managed .NET code.

### What it does
- loads assemblies
- reads metadata
- verifies code / type safety
- manages memory with garbage collection
- provides exception handling infrastructure
- supports threading, diagnostics, and profiling

### What it is not
- not your app
- not a user-facing program
- not the compiler (that is Roslyn, a separate build-time tool)

## 3) IL / CIL / MSIL
When you build a .NET app, the compiler usually generates:
- **CIL** (**Common Intermediate Language**)
- **metadata** about types, methods, and references

Common names:
- **IL** = normal developer term
- **CIL** = formal term
- **MSIL** = older Microsoft term

### Why IL exists
IL is:
- CPU-independent
- portable across supported platforms
- meant to be converted later into native code

## 4) JIT
**JIT (Just-In-Time compiler)** converts **IL -> native machine code** when code is needed.

Important clarification:
- JIT is **not a separate app**
- JIT is a **component inside the CLR / CoreCLR**

## 5) Execution Flow
```text
Source code
 -> compiler
 -> IL/CIL + metadata
 -> CLR / CoreCLR loads it
 -> JIT compiles needed methods
 -> native machine code
 -> CPU executes
```

## 6) Easy Distinctions
- **Compiler (Roslyn)**: build-time app/tool, source code -> IL; separate from CLR; delivered via .NET SDK
- **IL**: portable intermediate code
- **CLR**: runtime that executes managed code
- **JIT**: CLR component that compiles IL to native code
- **GC**: CLR subsystem that manages memory

## 7) CLR vs CoreCLR
- **CLR** = broad/common term
- **CoreCLR** = modern runtime implementation for mainstream modern .NET desktop/server workloads

Practical rule:
For ASP.NET Core, console apps, workers, and many Azure-hosted .NET apps, think **CoreCLR**.

## 8) Exceptions
### Normal app exceptions
Examples:
- `NullReferenceException`
- `InvalidOperationException`
- `ArgumentException`

These are regular managed exceptions:
- usually catchable with `try/catch`
- part of normal CLR exception flow

### Rare severe failures
If process/runtime state is corrupted (for example by unsafe code or bad native interop), the outcome is often:
- process crash
- no safe recovery
- not something normal `try/catch` should be expected to fix

## 9) Does CLR keep the OS safe?
Best phrasing:
- CLR makes **managed apps safer** by enforcing managed execution rules and reducing many memory-corruption risks
- the **OS itself** is still protected by operating-system boundaries, permissions, and process isolation

## 10) Azure
### Short answer
**Many Azure apps use CLR, but not all.**

Azure is a hosting platform, not one runtime. It can host:
- .NET
- Java
- Node.js
- Python
- Go
- containers with any runtime inside

### If the app is .NET
Then yes: it normally runs on the .NET runtime, and in modern mainstream cases that usually means **CoreCLR**.

Examples:
- ASP.NET Core on Azure App Service
- .NET Web API on App Service
- C# Azure Functions
- .NET worker services

### If the app is not .NET
Then CLR is not involved.
Examples:
- Java -> JVM
- Node.js -> Node/V8
- Python -> CPython

## 11) App Service and Functions
### App Service
If you deploy an ASP.NET Core app to Azure App Service, Azure hosts the app and the app runs using a supported .NET runtime on that host.

### Azure Functions
For C# Azure Functions, your code runs on .NET.
Execution model matters:
- **Isolated worker**: code runs in a separate .NET worker process
- **In-process**: code runs in the same process as the Functions host

## 12) Best Short Answers
- **What compiles source code to IL?** -> **The compiler (Roslyn)** — a build-time tool, part of .NET SDK
- **Is the compiler part of CLR?** -> **No** — it is a separate build-time executable
- **Who delivers the compiler?** -> **Microsoft** (Roslyn is open-source)
- **Is the compiler OS-specific?** -> Same source, but OS-specific binaries; cross-compilation is supported
- **What understands .NET code, runs it, and manages memory?** -> **CLR**
- **What turns IL into machine code?** -> **JIT**
- **Is JIT a separate app?** -> **No, it is part of CLR/CoreCLR**
- **Is CLR an app?** -> **No, it is the runtime/execution engine**
- **Do .NET apps on Azure use CLR?** -> **Yes, but Azure can host many non-.NET runtimes too**

## 13) One precise sentence
> In .NET, source code is compiled to IL and metadata; at runtime, the CLR (usually CoreCLR in modern .NET) loads that code, manages memory and runtime services, and uses JIT to turn IL into native machine code.

## 14) Official sources
- Managed execution process: https://learn.microsoft.com/en-us/dotnet/standard/managed-execution-process
- CLR overview: https://learn.microsoft.com/en-us/dotnet/standard/clr
- Runtime compilation options: https://learn.microsoft.com/en-us/dotnet/core/runtime-config/compilation
- Azure App Service (.NET): https://learn.microsoft.com/en-us/azure/app-service/configure-language-dotnetcore
- Azure Functions runtime versions: https://learn.microsoft.com/en-us/azure/azure-functions/functions-versions
