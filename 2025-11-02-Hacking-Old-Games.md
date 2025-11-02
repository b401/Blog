---
title: Hacking Old Games Part 1
author: i4
date: M11-02-2025
---

## Hello friends!

Back after two years with some cool stuff!  
I’ve been poking at reverse engineering again and building some small tools with a friend.

In this series we’ll look at reverse engineering and writing hacks for a game released in 1999 - [Well of Souls](http://www.synthetic-reality.com/wosHome.htm).  
Despite its age it still packs a few anti‑debug and anti‑tamper tricks, which makes it a nice playground.

Tools used in this post:
- Binary Ninja
- x32dbg


The following tools are being used in this post:
- Binary Ninja
- x32dbg

> Educational intent: This material is for learning reverse engineering concepts. Respect licenses and applicable laws before modifying software you do not own. :>)

## Anti-Debugging

We start out by attaching x32dbg (or any debugger of your choosing) to the `souls.exe` process and create a wizard called Gandalf.

![wizard](/b/images/wizard.png)

Once we incarnate to the world though, the game freezes and we see the following exception in x32dbg:

![wizard](/b/images/souls_wizard_freeze.png)

Interestingly, this does not happen if the debugger is not attached.

Old titles often rely on well-known Win32 APIs rather than custom obfuscation. One good suspect: `IsDebuggerPresent`.

There are some common ways applications can easily detect if a debugger has been attached. If we consider the games age, it will probably use something simple and not a home-brew anti-debugging technique.

We run it once again and attach x32dbg. This time, we change the view inside x32dbg to modules:

![loaded_modules](/b/images/x32dbg_modules.png)

After selecting all loaded modules, we can search for imports that have this name and sort by ordinal (a numeric reference to an exported function that is being imported).

![isdebuggerpresent](/b/images/isdebuggerpresent_module.png)

We have two hits! Which means that `souls.exe` makes use of `IsDebuggerPresent`. Now the question is where exactly does this get referenced in the code. To figure this out, we have to jump into a disassembler (IDA/Ghidra/Binary Ninja). 

But in static analysis (Binary Ninja here) there is no direct import entry. So it must be resolved dynamically.

To figure this out, we set a breakpoint on the runtime call inside `kernel32.dll` (which forwards to `kernelbase.dll` on modern Windows) and let it run.

![isdebuggerpresent](/b/images/break_on_debugger.png)

And we break on the call. 
![isdebuggerpresent](/b/images/break_on_debugger_call.png)

When a `call` executes, the CPU pushes the return address (next instruction) onto the stack (ESP on 32-bit). Grabbing that lets us jump back to the caller in the disassembler.


![calling stack](/b/images/calling_stack.png)

![return address](/b/images/return_address.png)

The binary was built without ASLR, so the address is stable run to run (verified via PE header flags).  Which means we can jump to this address inside our disassembler.

![dynamic loading](/b/images/dynamic_loading.png)

We see `LoadLibraryA` followed by a `GetProcAddress` pattern, but both the DLL name and the function name come indirectly from a helper (renamed mentally here as `GetItemFromList`) that indexes into a data array (`data_5006d0`).  
Indices:
- 42 (0x2A) → `"IsDebuggerPresent"`
- 43 (0x2B) → `"kernel32.dll"`

So: numeric index -> string -> dynamic load/resolve -> call. No static import footprint.

![dynamic loading](/b/images/return_item_from_list.png)

We don't really care yet for the the function `sub_489f12` and `sub_489ef2`. The most important part is the item reference for the array `data_5006d0`

```C
strcpy(_Destination: arg2, _Source: (&data_5006d0)[eax])
```

We know that `eax` contains the first argument of the function call (43 and 42).
So by looking manually at the array, we should be able to find `kernel32.dll` at place 43 (0x2b) and `IsDebuggerPresent` at 42 (0x2a).

![dynamic loading](/b/images/array_deref.png)

Alright, we know how it’s loaded. How do we bypass it now?

## Evasion

The easiest way, is to use a `x32dbg`/`x64dbo` plugin called [ScyllaHide](https://github.com/x64dbg/ScyllaHide) but that only helps you for dynamic analysis but we want a passive bypass that works even with no debugger attached

There is also another issue I will go into more detail during a later blog post.
Wells of Soul employs anti-tampering technique. directly patching game code is (for now) blocked by anti‑tamper checks, so we patch the API implementation instead. On modern Windows, `IsDebuggerPresent` is forwarded through `kernel32.dll` to `kernelbase.dll`, so patching inside `kernelbase.dll` is reliable.

We overwrite the first bytes with a tiny stub that forces a FALSE return (`xor eax, eax ; ret` on 32-bit). That short‑circuits the check.


```C
// We patch IsDebuggerPresent to always return 0
// You can checkout the overwriten function in the module
// xor eax, eax
// ret
void overwrite_IsDebuggerPresent() {
  HMODULE kernelbase_handle = GetModuleHandleA("kernelbase.dll");
  if (kernelbase_handle) {
    FARPROC f_IsDebuggerPresent = GetProcAddress(kernelbase_handle, "IsDebuggerPresent");
    if(f_IsDebuggerPresent) {
      DWORD oldProtect = {0};
      VirtualProtect(f_IsDebuggerPresent, sizeof(void*), PAGE_EXECUTE_READWRITE, &oldProtect);
      unsigned char patch[] = { 0x31, 0xC0, 0xC3 }; // xor eax, eax / ret
      memcpy((void*)f_IsDebuggerPresent, patch, sizeof(patch));
      VirtualProtect(f_IsDebuggerPresent, sizeof(void*),oldProtect, &oldProtect);
    }
  }
}
```

This function gets called either through a Shim dll or after a dll injection in the same memory space of `souls.exe`.

First we get the address of the already loaded `kernelbase.dll` (not `kernel32.dll` since it references the `kernelbase.dll` and I had trouble overwriting inside `kernel32.dll`)
```c
// Get module handle of loaded kernelbase
  HMODULE kernelbase_handle = GetModuleHandleA("kernelbase.dll");
```

Afterwards we get a reference to the memory address of `IsDebuggerPresent` and change the memory permissions to `PAGE_EXECUTE_READWRITE`

```C
...
FARPROC f_IsDebuggerPresent = GetProcAddress(kernelbase_handle, "IsDebuggerPresent");
...
VirtualProtect(f_IsDebuggerPresent, sizeof(void*), PAGE_EXECUTE_READWRITE, &oldProtect);
``` 

Lastly we overwrite the `IsDebuggerPresent` with a small amount of instructions to clear out `eax` and return earlier before the actual function process starts and change back the memory permissions.

```C
unsigned char patch[] = { 0x31, 0xC0, 0xC3 }; // xor eax, eax / ret
memcpy((void*)f_IsDebuggerPresent, patch, sizeof(patch));
VirtualProtect(f_IsDebuggerPresent, sizeof(void*),oldProtect, &oldProtect);
```

At the end this will change the code to look like this


![dynamic loading](/b/images/isdebuggerpresent_early.png)

Now the game will jump to the `IsDebuggerPresent` function, clear out `eax` and immediately return to the calling function. Skipping the debugger check.

### What's next?

Later posts will:
- Explore other anti-debug/anti-cheat mechanisms.
- Analyze tamper protection routines.
- Potentially :>) produce a keygen (credit to work by [0x3bb](https://3bb.io)).

Stay tuned.