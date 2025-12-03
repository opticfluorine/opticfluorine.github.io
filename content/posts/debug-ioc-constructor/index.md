+++
date = '2025-12-02T06:34:11-05:00'
title = 'Debugging Native Code Crashes in .NET Hosted Service Startup'
tags = ['Bugs', '.NET', 'Sovereign']
+++

Recently I ran across an interesting bug while adding an inventory system to [Sovereign Engine](https://sovereignengine.com). The client was crashing early during startup due to a segmentation fault, suggesting that there was a problem with how I was calling native code.

Firing up the client in gdb confirmed that the crash was due to native code:
```
(gdb) bt
#0  0x00007fbeac67413e in ImGui::GetFontSize() () from [...]/runtimes/linux-x64/native/cimgui.so
#1  0x00007fff7b768b44 in ?? ()
#2  0x000000000cabbc0c in ?? ()
#3  0x00007ffff76d2ef8 in ?? () from /usr/share/dotnet/shared/Microsoft.NETCore.App/9.0.6/libcoreclr.so
#4  0x00007fffffffa988 in ?? ()
#5  0x0000000000000000 in ?? ()
```
Okay, so it looks like the client is crashing because of a segfault that happens when we call `ImGui::GetFontSize()`. Good news, we know what the offending call is. Bad news, it's one of the most common native calls in the client and doesn't do much to help us narrow down a root cause.

Finally, there's one more interesting clue: the crash occurs before the various [`BackgroundService`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-10.0&tabs=net-cli) instances have started. In other words, the crash happens while the Dependency Injection (DI) framework is constructing its initial object graph. The question is, which object causes the crash when constructed at startup? Reviewing the changes since the last good commit, there's no obvious change that would cause an object to make the offending call during construction.

Time to break out the .NET debugger and try to track it down.

## Finding the Bad Constructor in the Dependency Graph

With a crash in managed code (e.g. an uncaught exception), finding the bad constructor would be easy: just set a breakpoint for the thrown uncaught exception and inspect the backtrace. With a segfault in native code, however, the debugger simply terminates with no further information when the offending call is made.

Knowing the identity of the offending call (`ImGui.GetFontSize()` from the managed bindings) means that we can set a breakpoint at every call and check which call causes the crash. This would be a quick and easy solution if the call was rare, but as previously mentioned, the Sovereign client makes a very large number of calls to `ImGui.GetFontSize()`. Worse, none of the calls obviously appear in a constructor. Since there are a large number of calls and no obvious way to narrow down the set of candidates, it would be very tedious to set and test breakpoints for every possible offending call. Can we do better?

The client's dependency graph currently contains around 450 objects that are instantiated at startup. Since the offending call is coming from GUI code, and GUI code tends to appear near the root of the dependency graph (it's not referenced outside of the renderer scenes), most objects in the graph are probably being constructed successfully before the crash - especially leaf objects that have no dependencies themselves. If we set a breakpoint in a leaf constructor, we should be able to suspend execution in the middle of constructing the object graph and work forward from that point.

Doing this, we find ourselves in the middle of a recursive process in the DI framework calling all of the constructors through a depth-first search of the dependency graph. This is a recursive cycle involving a handful of methods in `Microsoft.Extensions.DependencyInjection`. The key step is in `CallSiteRuntimeResolver.VisitConstructor(ConstructorCallsite, RuntimeResolverContext)` which iterates the object's dependencies before finally invoking the constructor through reflection. We know that we can easily reach this point for an arbitrary leaf object; can we go a step further and quickly identify the offending constructor?

The answer is yes - we can use a logging breakpoint immediately before the reflection call to log the called constructors in order. If we do this, then the offending constructor should be the last to be logged before the debugger crashes with a segfault. We add a logging breakpoint to the `Invoke()` call and set it to output `constructorCallSite.ImplementationType?.ToString()`, causing the debugger to log the full type name of every constructed object in order. Then we remove the other breakpoints and restart the debugger. 

Sure enough, we get a list of constructed objects that ends with `InventoryGui`. Aha! That's one of the new classes from the most recent changes. But it doesn't call `ImGui.GetFontSize()` anywhere... right? 

## The Bug

Actually, `InventoryGui` *does* call `ImGui.GetFontSize()` indirectly - by calling a static method when setting a `readonly` member at initialization time. ImGui and its fonts haven't been initialized yet, so the cimgui library dereferences a null pointer when trying to access the font data, and a segfault occurs. Moving the call to the `Render()` method (called once per frame, and only after ImGui is initialized) resolves the issue. 

On to the next bug.

---

### AI Disclaimer

No AI was used in the writing of this post.
