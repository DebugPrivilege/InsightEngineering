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

You can now see from the console that tracing of `lsass.exe` is working, and both the `.run` and `.out` files are being written to disk.

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

## How to trace a service?

With Shadow Stacks temporarily disabled, you can not only trace `lsass.exe` but also other services running on the machine. For a clean and formatted output of service names and their PIDs, you can use the following command:

```
C:\Users\Admin\Desktop\ttd>wmic service get Name,ProcessId /format:table
Name                                      ProcessId
AJRouter                                  0
ALG                                       0
AppIDSvc                                  0
Appinfo                                   1228
AppMgmt                                   0
AppReadiness                              0
AppVClient                                0
AppXSvc                                   5128
AssignedAccessManagerSvc                  0
AudioEndpointBuilder                      1384
Audiosrv                                  2132
autotimesvc                               0
AxInstSV                                  0
BDESVC                                    0
BFE                                       2680
BITS                                      0
BrokerInfrastructure                      616

<< SNIPPET >>
```

To **disable** Shadow Stacks temporarily, you can run the following command:

```
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\kernel" -Name "UserShadowStacksForceDisabled" -PropertyType DWORD -Value 1 -Force
```

Suppose you want to trace the Secondary Logon service; you can use the following command to find its PID.

```
C:\Users\Admin\Desktop\ttd>wmic service get Name,ProcessId /format:table | findstr "seclogon"
seclogon                                  1244
```

Finally, you can begin tracing this service, and you’ll see that it works as expected.

```
C:\Users\Admin\Desktop\ttd>TTD.exe -out c:\traces -attach 1244
Microsoft (R) TTD 1.01.11 x64
Release: 1.11.429.0
Copyright (C) Microsoft Corporation. All rights reserved.


Attaching to 1244
    Initializing the recording of process (PID:1244) on trace file: c:\traces\svchost01.run
    Recording has started of process (PID:1244) on trace file: c:\traces\svchost01.run
(x64) (PID:1244): Recording stopped after 41359ms
  Full trace dumped to c:\traces\svchost01.run


Note: c:\traces\svchost01.out may contain personally identifiable or security related information,
including but not necessarily limited to file paths, registry, memory or file contents. Exact
information depends on target process activity while it was recorded. Please be aware of this
when sharing with other people.

c:\traces\svchost01.out contains additional information about the recording session.
```

![image](https://github.com/user-attachments/assets/98ac4b6c-a74c-4259-997f-b346829b4616)


To **re-enable Shadow Stack**, you can reverse the changes made to disable it: (This requires a reboot as well)

```
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\kernel" -Name "UserShadowStacksForceDisabled" -Value 0
```
