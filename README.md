# DInvoke
Dynamic replacement for PInvoke on Windows. DInvoke contains powerful primitives that may be combined intelligently to dynamically invoke unmanaged code from disk or from memory with careful precision. This may be used for many purposes such as PE parsing, intelligent dynamic API resolution, dynamically loading PE plugins at runtime, process injection, and avoiding API hooks.

Features:
* Dynamically invoke unmanaged APIs without PInvoke
* Primitives allowing for strategic API hook evasion 
* Manually map unmanaged PE modules from managed code
* Map PE modules into sections backed by arbitrary modules on disk
* Modular process injection API
* Growing library of data structures, delegates, and function wrappers (please share :-)
* .NET v3.5+ support

Conference talk (Staying # & Bringing Covert Injection Tradecraft to .NET): https://www.youtube.com/watch?v=FuxpMXTgV9s

Blog posts:

1. Emulating Covert Operations - Dynamic Invocation (Avoiding PInvoke & API Hooks): https://thewover.github.io/Dynamic-Invoke/
2. Coming soon.

This project was originally created for SharpSploit (https://github.com/cobbr/SharpSploit). With permission from the author(s), it is not hosted here as a standalone library and NuGet.

# Example
In the example below, we first call OpenProcess normally using PInvoke. Then, we will call it in several ways using DInvoke to demonstrate that each mechanism successfully executes the unmanaged code and evades API hooks.

```csharp

///Author: TheWover
using System;
using System.Runtime.InteropServices;

using Data = DInvoke.Data;
using DynamicInvoke = DInvoke.DynamicInvoke;
using ManualMap = DInvoke.ManualMap;

namespace SpTestcase
{
    class Program
    {

        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern IntPtr OpenProcess(
            Data.Win32.Kernel32.ProcessAccessFlags processAccess,
            bool bInheritHandle,
            uint processId
        );

        static void Main(string[] args)
        {
            // Details
            String testDetail = @"
            #=================>
            # Hello there!
            # I demonstrate API Hooking bypasses
            # by calling OpenProcess via
            # PInvoke then DInvoke.
            # All handles are requested with
            # PROCESS_ALL_ACCESS permissions.
            #=================>
            ";
            Console.WriteLine(testDetail);

            //PID of current process.
            uint id = Convert.ToUInt32(System.Diagnostics.Process.GetCurrentProcess().Id);

            //Process handle
            IntPtr hProc;

            // Create the array for the parameters for OpenProcess
            object[] paramaters =
            {
                Data.Win32.Kernel32.ProcessAccessFlags.PROCESS_ALL_ACCESS,
                false,
                id
            };

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

            //////////////////////////////////////////////////////////////////////////////////////////////////////////
            // Call OpenProcess using PInvoke
            Console.WriteLine("[?] Call OpenProcess via PInvoke ...");
            hProc = OpenProcess(Data.Win32.Kernel32.ProcessAccessFlags.PROCESS_ALL_ACCESS, false, id);
            Console.WriteLine("[>] Process handle : " + string.Format("{0:X}", hProc.ToInt64()) + "\n");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

            //////////////////////////////////////////////////////////////////////////////////////////////////////////
            // Call OpenProcess using GetLibraryAddress (underneath the hood)
            Console.WriteLine("[?] Call OpenProcess from the loaded module list using System.Diagnostics.Process.GetCurrentProcess().Modules ...");
            hProc = DynamicInvoke.Win32.OpenProcess(Data.Win32.Kernel32.ProcessAccessFlags.PROCESS_ALL_ACCESS, false, id);
            Console.WriteLine("[>] Process handle : " + string.Format("{0:X}", hProc.ToInt64()) + "\n");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

            //////////////////////////////////////////////////////////////////////////////////////////////////////////
            // Search function by name from module in PEB
            Console.WriteLine("[?] Specifying the name of a DLL (\"kernel32.dll\"), search the PEB for the loaded module and resolve a function by walking the export table in-memory...");
            Console.WriteLine("[+] Search by name --> OpenProcess");
            IntPtr pkernel32 = DynamicInvoke.Generic.GetPebLdrModuleEntry("kernel32.dll");
            IntPtr pOpenProcess = DynamicInvoke.Generic.GetExportAddress(pkernel32, "OpenProcess");

            //Call OpenProcess
            hProc = (IntPtr)DynamicInvoke.Generic.DynamicFunctionInvoke(pOpenProcess, typeof(DynamicInvoke.Win32.Delegates.OpenProcess), ref paramaters);
            Console.WriteLine("[>] Process Handle : " + string.Format("{0:X}", hProc.ToInt64()) + "\n");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

            //////////////////////////////////////////////////////////////////////////////////////////////////////////
            // Manually map kernel32.dll
            // Search function by name from module in PEB
            Console.WriteLine("[?] Manually map a fresh copy of a DLL (\"kernel32.dll\"), and resolve a function by walking the export table in-memory...");
            Console.WriteLine("[+] Search by name --> OpenProcess");
            Data.PE.PE_MANUAL_MAP moduleDetails = ManualMap.Map.MapModuleToMemory("C:\\Windows\\System32\\kernel32.dll");
            Console.WriteLine("[>] Module Base : " + string.Format("{0:X}", moduleDetails.ModuleBase.ToInt64()) + "\n");

            //Call OpenProcess
            hProc = (IntPtr)DynamicInvoke.Generic.CallMappedDLLModuleExport(moduleDetails.PEINFO, moduleDetails.ModuleBase, "OpenProcess", typeof(DynamicInvoke.Win32.Delegates.OpenProcess), paramaters);
            Console.WriteLine("[>] Process Handle : " + string.Format("{0:X}", hProc.ToInt64()) + "\n");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

            //////////////////////////////////////////////////////////////////////////////////////////////////////////
            // Map kernel32.dll using Module Overloading
            // Search function by name from module in PEB
            Console.WriteLine("[?] Use Module Overloading to map a fresh copy of a DLL (\"kernel32.dll\") into memory backed by another file on disk. Resolve a function by walking the export table in-memory...");
            Console.WriteLine("[+] Search by name --> OpenProcess");
            moduleDetails = ManualMap.Overload.OverloadModule("C:\\Windows\\System32\\kernel32.dll");
            Console.WriteLine("[>] Module Base : " + string.Format("{0:X}", moduleDetails.ModuleBase.ToInt64()) + "\n");

            //Call OpenProcess
            hProc = (IntPtr)DynamicInvoke.Generic.CallMappedDLLModuleExport(moduleDetails.PEINFO, moduleDetails.ModuleBase, "OpenProcess", typeof(DynamicInvoke.Win32.Delegates.OpenProcess), paramaters);
            Console.WriteLine("[>] Process Handle : " + string.Format("{0:X}", hProc.ToInt64()) + "\n");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

            //////////////////////////////////////////////////////////////////////////////////////////////////////////
            Console.WriteLine("[!] Test complete!");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();

        }
    }
}


```

To test that this evaded hooks, we will use the tool `API Monitor v2` to hook `kernel32.dll!OpenProcess`. Then we will run the demo through API Monitor. You may observe which of our calls to OpenProcess were caught in hooks by watching for those that are called with the `PROCESS_ALL_ACCESS` flag. As you will see, API Monitor successfully catches the API call when it is performed with PInvoke. However, it does NOT succeed when we use DInvoke or Manual Mapping. You may watch the video on Vimeo to see this in action.

[SharpSploit: Bypassing API Hooks via DInvoke and Manual Mapping](https://player.vimeo.com/video/406589341)

# Credit
* The Wover
* FuzzySec (b33f)
* cobbr
