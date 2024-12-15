# Summary

This page covers common troubleshooting issues related to TTD itself. It’s a work in progress and will be updated as new common issues are identified.

## How to trace lsass.exe?

A common issue that users might encounter is an error that appears when attempting to trace `lsass.exe`:

```
C:\Users\Admin\Desktop\ttd>TTD.exe -out c:\traces -attach 896
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Attaching to 896
Error:  Error (internal): It looks like launching/attaching to the guest encountered an unexpected error (Error Code 0x80004005:  Unspecified error)
Error:  An error occurred (Error Code 0x80004005:  Unspecified error)
Error:  Failed to start recording process 896 (Error Code 0x80004005:  Unspecified error)
Error:  Recording of  (PID: 896) did not complete successfully
Error:  Corrupted trace file written to 'c:\traces\lsass01.run.err'.

Error text from c:\traces\lsass01.out (see file for full details):

  Error: Process with Id 896 has shadow stacks enforcement enabled and therefore is not supported for recording
Error:  Failed to attach to the guest process; output file is 'c:\traces\lsass01.out'
```

To resolve this, Shadow Stacks need to be temporarily disabled by setting the following registry value. Note that a reboot is required for the changes to take effect:

```
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\kernel" -Name "UserShadowStacksForceDisabled" -PropertyType DWORD -Value 1 -Force
```

You can now see from the console that tracing of lsass.exe is working, and both the .run and .out files are being written to disk.

```
C:\Users\Admin\Desktop\ttd>TTD.exe -out c:\traces -attach 904
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Attaching to 904
    Initializing the recording of process (PID:904) on trace file: c:\traces\lsass03.run
    Recording has started of process (PID:904) on trace file: c:\traces\lsass03.run
(x64) (PID:904): Recording stopped after 19140ms
  Full trace dumped to c:\traces\lsass03.run


Note: c:\traces\lsass03.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\lsass03.out contains additional information about the recording session.
```

Another way to verify is by using a tool like `ProcMon` or `SystemInformer` to check if `TTDRecordCPU.dll` has been successfully loaded into `lsass.exe`.

![image](https://github.com/user-attachments/assets/cad04764-c4ee-426b-8263-7884d2139e24)

To **re-enable Shadow Stack**, you can reverse the changes made to disable it: (This requires a reboot as well)

```
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\kernel" -Name "UserShadowStacksForceDisabled" -Value 0
```
