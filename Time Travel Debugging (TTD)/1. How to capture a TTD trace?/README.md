# Summary

Let's start by examining the various command-line arguments supported by TTD, understanding their functions, and exploring their usage.

```
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


TTD (Time Travel Debugging) is a tool that allows you to capture a trace of your
process as it executes into a file. Once you have the trace file you can debug
the process yourself in Windbg (https://aka.ms/windbg) or give the file to someone
else to debug, as if it was a live process. In addition, TTD enables backwards
execution, allowing you to time travel back to the source of a problem.

Usage: TTD.exe [options] [mode] [program [<arguments>]]

Options:
 -?                Display this help.
 -help             Display this help.
 -acceptEula       Accept one-time EULA without prompting.
 -children         Trace through family of child processes.
 -cleanup          Uninstall process monitor driver
 -cmdLineFilter "<string>"
                   Must be combined with -monitor and it will only
                   record the target if its command line contains the string.
                   This is useful for situations when the command line argument
                   uniquely identifies the process you are interested in,
                   e.g., notepad.exe specialfile.txt only the instance of
                   notepad.exe with that file name will be recorded.
 -maxConcurrentRecordings <count>
                   Maximum number of recordings that can be ongoing at any one
                   point in time.
 -maxFile <size>   Maximum size of the trace file in MB.  When in full trace
                   mode the default is 1024GB and the minimum value is 1MB.
                   When in ring buffer mode the default is 2048MB, the minimum
                   value is 1MB, and the maximum value is 32768MB.
                   The default for in-memory ring on 32-bit processes is 256MB.
 -module <module>  Restrict recording to specified module. More than one -module
                   may be specified. The module name is specified without a path,
                   such as notepad.exe or shell32.dll.
 -noUI             Disables the UI for manual control of recording.
 -numVCpu <number> Specifies a number of Virtual CPUs to be reserved and used
                   when tracing. This value affects the total memory overhead
                   placed on the guest process' memory by TTD. If not
                   specified then default per platform is used: 55 for x64/ARM64
                   and 32 for x86. Change this setting in order to
                   limit the memory impact ONLY if you are running out of
                   memory. The .out file will give hints this effect.
                   Note: Changing this value to a lower number can severely
                         impact the performance of tracing and should only
                         be done to work around memory impact issues.
 -onInitCompleteEvent <eventName>
                   Allows an event to be signaled when tracing initialization
                   is complete.
 -onMonitorReadyEvent <eventName>
                   Allows an event to be signaled when the recording monitor is ready
                   to record new processes.
 -out <file>       Specify a trace file name or a directory.  If a directory, the
                   directory must already exist. If a file name, the directory
                   of the file name must already exist. If a file name, and the file
                   name already exists, it will be overwritten without prompting. By
                   default the executable's base name with a version number is used
                   to prefix the trace file name.
 -passThroughExit  Pass the guest process exit value through as
                   TTD.exe's exit value.
 -recordMode <mode> Specifies how recording is triggered in the target process. Possible modes:
                   Automatic - Recording occurs automatically, according to the other
                               specified options (default).
                   Manual    - Recording is disabled to start, but can be turned on/off
                               via API calls.
 -replayCpuSupport <support>  Specifies what support is expected from
                   the CPUs that will be used to replay the trace.
                   Possible <support> values:
                   Default           - Default CPU support, just requires basic
                                       commonly-available support in the replay CPU.
                   MostConservative  - Requires no special support in the replay CPU.
                                       Adequate for traces that will be replayed on a
                                       completely different CPU architecture,
                                       like an Intel trace on ARM64.
                   MostAggressive    - Assumes that the replay CPU will be similar
                                       and of equal or greater capability
                                       than the CPU used to record.
                   IntelAvxRequired  - Assumes that the replay CPU will be
                                       Intel/AMD 64-bit CPU supporting AVX.
                   IntelAvx2Required - Assumes that the replay CPU will be
                                       Intel/AMD 64-bit CPU supporting AVX2.
 -ring             Trace to a ring buffer. The file size will not grow beyond
                   the limits specified by -maxFile. Only the latter portion of the
                   recording that fits within the given size will be preserved.
 -timestampFileName
                   When choosing a name for the .run file use current time
                   as part of the file name, instead of an increasing counter.
                   When repeatedly recording the same executable, use this
                   switch to reduce time to start tracing.
 -tracingOff       Starts application with trace recording off.  You can
                   use the UI to turn tracing on.

Modes:
 -attach <PID>      Attach to a running process specified by process ID.
 -launch            Launch and trace the program (default).
                    This is the only mode that allows arguments to be passed
                    to the program.
                    Note: This must be the last option in the command-line,
                    followed by the program + <arguments>
 -monitor <Program> Trace programs or services each time they are started
                    (until reboot or Ctrl+C pressed). You must specify a full
                    path to the output location with -out. More than one
                    -monitor switch may be specified.

Control:
 -stop             Stop tracing the process specified (name, PID or "all").
 -wait <timeout>   Wait for up to the amount of seconds specified for all
                   trace sessions on the system to end.  Specify -1 to wait
                   infinitely.

Examples:

--- Example 1: Launch a program with arguments and record it ---

    TTD.exe ping.exe msn.com

    Records 'ping.exe msn.com' to .run file in current directory.

--- Example 2: Attach to a process and record it to a directory ---

    md c:\traces  (ensure c:\traces directory exists)
    TTD.exe -out c:\traces -attach 1234

    Attaches to PID 1234 (decimal) and records .run to c:\traces.

--- Example 3: Monitor for program launch and record it to a directory ---

    md c:\traces  (ensure c:\traces directory exists)
    TTD.exe -out c:\traces -monitor notepad.exe

    Starts recording of any notepad.exe already running and enters a loop
    waiting for future launches of notepad.exe. All trace files are written
    to c:\traces. Press Ctrl+C to exit the monitoring loop.

--- Example 4: Record a program with upper limit on trace file size ---

    TTD.exe -ring -maxfile 8192 -attach 1234

    Attaches to PID 1234 (decimal) and records .run to current directory.
    The .run file will not exceed 8192MB, overwriting previously recorded
    information if necessary.

--- Example 5: Controlling how the .run file name is chosen ---

    By default a sequential scan is done to find an unused file in the
    output directory. If ping.exe is recorded the recorder will try ping01.run,
    ping02.run, etc. until an unused file name is found. For most scenarios
    this naming method is sufficient.

    However, if you want to record the same program many times the default
    file naming algorithm can become expensive. An alternative method can
    be used in this scenario:

    TTD.exe -out c:\traces -timestampFileName ping.exe msn.com

    When -timestampFileName is used the file name looks more like ping_2023-06-17_103116.run.
    The odds are very small of a collision with this naming method (resolution of one
    second) but the recorder will repeat the process until a unique file name is found.


--- Example 6: Limiting scope of recording to specified module(s) ---

    By default TTD records all user mode execution in the target, which can
    result in substantial slowdowns in execution time and large trace file
    size.

    With the -module option you have the ability to focus TTD on the module(s)
    that you care about. TTD will start recording when execution enters the module
    and stop recording when execution leaves the module. This can result in
    substantially faster recording and smaller trace files. For example, if
    you are only interested in recording msvcrt.dll while ping.exe is running
    you can use the following command:

    TTD.exe -out c:\traces -module msvcrt.dll ping.exe msn.com

    Note that any calls the listed module(s) make to other modules are also
    recorded (e.g., if msvcrt.dll calls kernel32.dll, then kernel32.dll will
    also be recorded, but only when kernel32.dll is called by msvcrt.dll). This
    is done to mitigate performance impact when modules call short functions in
    other modules.

    You can specify more than one module to record by using multiple -module
    switches.
```

# Common Tracing Options

This section will cover the most commonly used tracing options. While the TTD.exe documentation provides sufficient details about the command-line arguments, we will review the most common ones to provide additional context and make them even easier to understand.

**--- Example 1: Launch and record a setup program to a specific directory ---**

This command launches `VmManagedSetup.exe` and records its execution into a `.run` trace file saved in the `c:\traces` directory.

```
TTD.exe  -out c:\traces C:\temp\VmManagedSetup.exe
```

When tracing is complete or manually stopped, the console will display the final output, and both a `.run` file and an `.out` file will be saved in the specified directory.

```
C:\Users\Admin\Desktop\ttd>TTD.exe  -out c:\traces C:\temp\VmManagedSetup.exe
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Launching 'C:\temp\VmManagedSetup.exe'
    Initializing the recording of process (PID:8768) on trace file: c:\traces\VmManagedSetup01.run
    Recording has started of process (PID:8768) on trace file: c:\traces\VmManagedSetup01.run
VmManagedSetup.exe(x64) (PID:8768): Process exited with exit code 1 after 211671ms
  Full trace dumped to c:\traces\VmManagedSetup01.run


Note: c:\traces\VmManagedSetup01.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\VmManagedSetup01.out contains additional information about the recording session.
```

In this example, the size of the trace file is 40 MB.

```
PS C:\Users\Admin> Get-Item "C:\traces\VmManagedSetup01.run" | Select-Object Name, @{Name="Size (MB)"; Expression={[math]::Round($_.Length / 1MB, 2)}}

Name                 Size (MB)
----                 ---------
VmManagedSetup01.run        40
```

In this example, I chose to set breakpoints on specific function calls of interest within this particular sample.

![image](https://github.com/user-attachments/assets/772c63ed-fa24-4d0d-9d21-b864f5fc6432)


**--- Example 2: Launch a program and record its execution along with child processes ---**

This command runs `Basta.exe` from the `C:\temp` directory and records its activity in a `.run` file saved in `c:\traces`. The `-children` option also captures any child processes started by `Basta.exe` and creates separate `.run` files for each one in the same folder.

```
TTD.exe -Children -out c:\traces C:\temp\Basta.exe
```

The output shows that `Basta.exe` and its child processes `cmd.exe` and `vssadmin.exe` were successfully recorded, with separate `.run` trace files created for each process in the `c:\traces` directory. Each process exited, and the associated `.out` files contain additional session details.

```
C:\Users\WDAGUtilityAccount\Desktop\ttd>TTD.exe -Children -out c:\traces C:\temp\Basta.exe
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Launching 'C:\temp\Basta.exe'
    Initializing the recording of process (PID:3676) on trace file: c:\traces\Basta01.run
    Recording has started of process (PID:3676) on trace file: c:\traces\Basta01.run
    Initializing the recording of process (PID:2308) on trace file: c:\traces\cmd01.run
    Recording has started of process (PID:2308) on trace file: c:\traces\cmd01.run
    Initializing the recording of process (PID:4320) on trace file: c:\traces\vssadmin01.run
    Recording has started of process (PID:4320) on trace file: c:\traces\vssadmin01.run
(x64) (PID:4320): Process exited with exit code 2 after 156ms
  Trace family nesting level is 2; Parent process ID is 2308
  Full trace dumped to c:\traces\vssadmin01.run


Note: c:\traces\vssadmin01.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\vssadmin01.out contains additional information about the recording session.

(x86) (PID:2308): Process exited with exit code 2 after 531ms
  Trace family nesting level is 1; Parent process ID is 3676
  Full trace dumped to c:\traces\cmd01.run


Note: c:\traces\cmd01.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\cmd01.out contains additional information about the recording session.

Basta.exe(x86) (PID:3676): Process exited with exit code 0 after 150531ms
  Full trace dumped to c:\traces\Basta01.run


Note: c:\traces\Basta01.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\Basta01.out contains additional information about the recording session.
```

**--- Example 3: Launch a program and record only specific module activity ---**

This command launches `VmManagedSetup.exe` and records its execution into a `.run` file saved in `c:\traces`, focusing only on code execution within the `ws2_32.dll` module. This limits the recording to network-related operations, which will reduce trace size.

```
TTD.exe -out c:\traces -module ws2_32.dll C:\Temp\VmManagedSetup.exe
```

When tracing is complete or manually stopped, the console will display the final output, and both a `.run` file and an `.out` file will be saved in the specified directory. However, you will notice that the size of the trace file is much smaller than the first example.

```
C:\Users\Admin\Desktop\ttd>TTD.exe -out c:\traces -module ws2_32.dll C:\Temp\VmManagedSetup.exe
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Launching 'C:\Temp\VmManagedSetup.exe'
    Initializing the recording of process (PID:5660) on trace file: c:\traces\VmManagedSetup02.run
    Recording has started of process (PID:5660) on trace file: c:\traces\VmManagedSetup02.run
VmManagedSetup.exe(x64) (PID:5660): Process exited with exit code 0 after 141157ms
  Full trace dumped to c:\traces\VmManagedSetup02.run


Note: c:\traces\VmManagedSetup02.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\VmManagedSetup02.out contains additional information about the recording session.
```

In this example, the size of the trace file is only 4M.

```
PS C:\Users\Admin> Get-Item "C:\traces\VmManagedSetup02.run" | Select-Object Name, @{Name="Size (MB)"; Expression={[math]::Round($_.Length / 1MB, 2)}}

Name                 Size (MB)
----                 ---------
VmManagedSetup02.run         4
```

In this example, I decided to set breakpoints on specific function calls of interest within this sample. However, compared to the first example, you'll notice that the recorded calls are limited to functions within the `ws2_32.dll` module.

![image](https://github.com/user-attachments/assets/d790bcc6-c9f9-4fc8-9195-c9887a0bf304)


**--- Example 4: Attach to a running process and record its execution ---**

This command attaches TTD to the running process with `PID` 8372 and records its execution into a `.run` file saved in the `c:\traces` directory. The trace captures the process's user-mode execution from the moment of attachment.

```
C:\Users\Admin\Desktop\ttd>TTD.exe -out c:\traces -attach 8372
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Attaching to 8372
    Initializing the recording of process (PID:8372) on trace file: c:\traces\VmManagedSetup03.run
    Recording has started of process (PID:8372) on trace file: c:\traces\VmManagedSetup03.run
(x64) (PID:8372): Process exited with exit code 0 after 144359ms
  Full trace dumped to c:\traces\VmManagedSetup03.run


Note: c:\traces\VmManagedSetup03.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\VmManagedSetup03.out contains additional information about the recording session.
```

**--- Example 5: Monitor a program for future launches and record each instance ---**

This command sets up TTD to watch for any instance of `VmManagedSetup.exe` being launched. When the program runs, TTD records its execution to a `.run` file in the `c:\traces` directory. It also creates a service called `ProcLaunchMon` and loads the `ProcLaunchMon.sys` driver to handle the monitoring, which stays active until stopped or the system is restarted.

```
C:\Users\Admin\Desktop\ttd>TTD.exe -out c:\traces -monitor VmManagedSetup.exe
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Successfully uninstalled the Process Launch Monitor driver
Connecting to running processes, if any.
Initial connections complete.
Successfully installed the Process Launch Monitor driver
Tracking process consent.exe(6376)        From parent process svchost.exe(1184)
Tracking process TabTip.exe(6516)        From parent process svchost.exe(1344)
Tracking process TabTip.exe(5376)        From parent process svchost.exe(1344)
Recording process VmManagedSetup.exe(532)        From parent process explorer.exe(5680)
    Initializing the recording of process (PID:532) on trace file: c:\traces\VmManagedSetup04.run
    Recording has started of process (PID:532) on trace file: c:\traces\VmManagedSetup04.run
Tracking process explorer.exe(1540)        From parent process SystemInformer.exe(7052)
Tracking process explorer.exe(4864)        From parent process svchost.exe(80)
(x64) (PID:532): Recording stopped after 165063ms
  Full trace dumped to c:\traces\VmManagedSetup04.run


Note: c:\traces\VmManagedSetup04.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\VmManagedSetup04.out contains additional information about the recording session.
```
