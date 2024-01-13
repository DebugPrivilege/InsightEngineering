# What is Windows Error Reporting

Windows Error Reporting (WER) is a feature of Microsoft Windows that provides crash reporting solutions. It was introduced with Windows XP and has been included in subsequent versions of Windows. When a program crashes or encounters a critical error, WER collects debug information, which includes information about the state of the program at the time of the crash.

Here is an example of a log entry that records **beacon.exe** experiencing a problem, and Windows Error Reporting created a crash report for it:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/655978ce-70a9-4085-bb69-2a084fe639db)

The **attached files** contains a list of files that are prepared to be sent to Microsoft for analysis if consent was given. This includes the following:

- A minidump file **`(WER035.tmp.mdmp)`** which is a snapshot of the application's memory at the time of the crash.
- An internal metadata XML file **`(WER093.tmp.WERInternalMetadata.xml)`** that contains data about the crash report.
- A CSV file **`(WER0A2.tmp.csv)`** that contains additional data about processes running at the time of a system error or crash.
- Two text files **`WER0C2.tmp.txt`** and **`WER0E4.tmp.appcompat.txt`** which could contain text information about the crash and compatibility data.
- A heap dump file **`(memory.hdmp)`** that contains a dump of the application's heap, providing additional information about the application's state at the time of the crash.
- Windows Error Reporting (WER) file **`(Report.wer)`** that contains information about the error, such as the date time of the crash, the exception code, SHA1 hash of the application, the modules that were loaded, etc.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/82739c54-d4c1-49aa-a3f1-b602b3546503)

# WER Report

The **`Report.wer`** file is a Windows Error Reporting file that logs detailed information about an application crash. It includes event type, timestamps, consent for reporting, application name, version, modules loaded during the crash, OS version, SHA1 of the application, and system information like locale ID. Specific error details such as exception codes are also recorded, which indicate the type of crash (e.g., access violation). 

```
Version=1
EventType=BEX64
EventTime=133496477153022430
ReportType=2
Consent=1
UploadTime=133496506956013117
ReportStatus=16777220
ReportIdentifier=af591e90-8a5e-4a3c-a9f1-3ddaa5e0b04b
IntegratorReportIdentifier=db1268ee-55b9-4afb-b9d3-d29525606682
Wow64Host=34404
NsAppName=beacon.exe
AppSessionGuid=000029d8-0002-000d-45ec-78a85646da01
TargetAppId=W:0006d60e4d727c5556b492c242c0fa567c630000ffff!00000140ae80d8abddbb12582ce734016833b2871e36!beacon.exe
TargetAppVer=1970//01//01:00:00:00!543b0!beacon.exe
BootId=4294967295
HeapdumpAttached=1
TargetAsId=2458
IsFatal=1
EtwNonCollectReason=4
Response.type=4
Sig[0].Name=Application Name
Sig[0].Value=beacon.exe
Sig[1].Name=Application Version
Sig[1].Value=0.0.0.0
Sig[2].Name=Application Timestamp
Sig[2].Value=00000000
Sig[3].Name=Fault Module Name
Sig[3].Value=StackHash_0000
Sig[4].Name=Fault Module Version
Sig[4].Value=0.0.0.0
Sig[5].Name=Fault Module Timestamp
Sig[5].Value=00000000
Sig[6].Name=Exception Offset
Sig[6].Value=PCH_84
Sig[7].Name=Exception Code
Sig[7].Value=c0000005
Sig[8].Name=Exception Data
Sig[8].Value=0000000000000008
DynamicSig[1].Name=OS Version
DynamicSig[1].Value=10.0.19043.2.0.0.256.48
DynamicSig[2].Name=Locale ID
DynamicSig[2].Value=2057
DynamicSig[22].Name=Additional Information 1
DynamicSig[22].Value=0000
DynamicSig[23].Name=Additional Information 2
DynamicSig[23].Value=00000000000000000000000000000000
DynamicSig[24].Name=Additional Information 3
DynamicSig[24].Value=0000
DynamicSig[25].Name=Additional Information 4
DynamicSig[25].Value=00000000000000000000000000000000
UI[2]=C:\users\admin\desktop\beacon.exe
LoadedModule[0]=C:\users\admin\desktop\beacon.exe
LoadedModule[1]=C:\Windows\SYSTEM32\ntdll.dll
LoadedModule[2]=C:\Windows\System32\KERNEL32.DLL
LoadedModule[3]=C:\Windows\System32\KERNELBASE.dll
LoadedModule[4]=C:\Windows\System32\msvcrt.dll
LoadedModule[5]=C:\Windows\System32\ADVAPI32.dll
LoadedModule[6]=C:\Windows\System32\sechost.dll
LoadedModule[7]=C:\Windows\System32\RPCRT4.dll
LoadedModule[8]=C:\Windows\SYSTEM32\WININET.dll
LoadedModule[9]=C:\Windows\System32\WS2_32.dll
LoadedModule[10]=C:\Windows\SYSTEM32\CRYPTSP.dll
LoadedModule[11]=C:\Windows\system32\rsaenh.dll
LoadedModule[12]=C:\Windows\System32\bcrypt.dll
LoadedModule[13]=C:\Windows\SYSTEM32\CRYPTBASE.dll
LoadedModule[14]=C:\Windows\System32\bcryptPrimitives.dll
LoadedModule[15]=C:\Windows\SYSTEM32\SspiCli.dll
LoadedModule[16]=C:\Windows\system32\mswsock.dll
LoadedModule[17]=C:\Windows\SYSTEM32\iertutil.dll
LoadedModule[18]=C:\Windows\System32\combase.dll
LoadedModule[19]=C:\Windows\System32\ucrtbase.dll
LoadedModule[20]=C:\Windows\System32\shcore.dll
LoadedModule[21]=C:\Windows\System32\user32.dll
LoadedModule[22]=C:\Windows\System32\win32u.dll
LoadedModule[23]=C:\Windows\System32\GDI32.dll
LoadedModule[24]=C:\Windows\System32\gdi32full.dll
LoadedModule[25]=C:\Windows\System32\msvcp_win.dll
LoadedModule[26]=C:\Windows\System32\IMM32.DLL
LoadedModule[27]=C:\Windows\SYSTEM32\windows.storage.dll
LoadedModule[28]=C:\Windows\SYSTEM32\Wldp.dll
LoadedModule[29]=C:\Windows\System32\shlwapi.dll
LoadedModule[30]=C:\Windows\SYSTEM32\profapi.dll
LoadedModule[31]=C:\Windows\SYSTEM32\ondemandconnroutehelper.dll
LoadedModule[32]=C:\Windows\SYSTEM32\winhttp.dll
LoadedModule[33]=C:\Windows\SYSTEM32\kernel.appcore.dll
LoadedModule[34]=C:\Windows\SYSTEM32\IPHLPAPI.DLL
LoadedModule[35]=C:\Windows\SYSTEM32\WINNSI.DLL
LoadedModule[36]=C:\Windows\System32\NSI.dll
LoadedModule[37]=C:\Windows\SYSTEM32\urlmon.dll
LoadedModule[38]=C:\Windows\SYSTEM32\srvcli.dll
LoadedModule[39]=C:\Windows\SYSTEM32\netutils.dll
LoadedModule[40]=C:\Windows\System32\OLEAUT32.dll
OsInfo[0].Key=vermaj
OsInfo[0].Value=10
OsInfo[1].Key=vermin
OsInfo[1].Value=0
OsInfo[2].Key=verbld
OsInfo[2].Value=19043
OsInfo[3].Key=ubr
OsInfo[3].Value=1237
OsInfo[4].Key=versp
OsInfo[4].Value=0
OsInfo[5].Key=arch
OsInfo[5].Value=9
OsInfo[6].Key=lcid
OsInfo[6].Value=2057
OsInfo[7].Key=geoid
OsInfo[7].Value=242
OsInfo[8].Key=sku
OsInfo[8].Value=48
OsInfo[9].Key=domain
OsInfo[9].Value=0
OsInfo[10].Key=prodsuite
OsInfo[10].Value=256
OsInfo[11].Key=ntprodtype
OsInfo[11].Value=1
OsInfo[12].Key=platid
OsInfo[12].Value=10
OsInfo[13].Key=sr
OsInfo[13].Value=0
OsInfo[14].Key=tmsi
OsInfo[14].Value=9988
OsInfo[15].Key=osinsty
OsInfo[15].Value=2
OsInfo[16].Key=iever
OsInfo[16].Value=11.789.19041.0-11.0.1000
OsInfo[17].Key=portos
OsInfo[17].Value=0
OsInfo[18].Key=ram
OsInfo[18].Value=4096
OsInfo[19].Key=svolsz
OsInfo[19].Value=299
OsInfo[20].Key=wimbt
OsInfo[20].Value=0
OsInfo[21].Key=blddt
OsInfo[21].Value=191206
OsInfo[22].Key=bldtm
OsInfo[22].Value=1406
OsInfo[23].Key=bldbrch
OsInfo[23].Value=vb_release
OsInfo[24].Key=bldchk
OsInfo[24].Value=0
OsInfo[25].Key=wpvermaj
OsInfo[25].Value=0
OsInfo[26].Key=wpvermin
OsInfo[26].Value=0
OsInfo[27].Key=wpbuildmaj
OsInfo[27].Value=0
OsInfo[28].Key=wpbuildmin
OsInfo[28].Value=0
OsInfo[29].Key=osver
OsInfo[29].Value=10.0.19041.1237.amd64fre.vb_release.191206-1406
OsInfo[30].Key=buildflightid
OsInfo[31].Key=edition
OsInfo[31].Value=Professional
OsInfo[32].Key=ring
OsInfo[32].Value=Retail
OsInfo[33].Key=expid
OsInfo[34].Key=fconid
OsInfo[35].Key=containerid
OsInfo[36].Key=containertype
OsInfo[37].Key=edu
OsInfo[37].Value=0
File[0].CabName=minidump.mdmp
File[0].Path=WERE035.tmp.mdmp
File[0].Flags=539033600
File[0].Type=2
File[0].Original.Path=\\?\C:\ProgramData\Microsoft\Windows\WER\Temp\WERE035.tmp.mdmp
File[1].CabName=WERInternalMetadata.xml
File[1].Path=WERE093.tmp.WERInternalMetadata.xml
File[1].Flags=327682
File[1].Type=5
File[1].Original.Path=\\?\C:\ProgramData\Microsoft\Windows\WER\Temp\WERE093.tmp.WERInternalMetadata.xml
File[2].CabName=memory.csv
File[2].Path=WERE0A2.tmp.csv
File[2].Flags=65538
File[2].Type=5
File[2].Original.Path=\\?\C:\ProgramData\Microsoft\Windows\WER\Temp\WERE0A2.tmp.csv
File[3].CabName=sysinfo.txt
File[3].Path=WERE0C2.tmp.txt
File[3].Flags=65538
File[3].Type=5
File[3].Original.Path=\\?\C:\ProgramData\Microsoft\Windows\WER\Temp\WERE0C2.tmp.txt
File[4].CabName=AppCompat.txt
File[4].Path=WERE0E4.tmp.appcompat.txt
File[4].Flags=16842754
File[4].Type=5
File[4].Original.Path=\\?\C:\Users\Admin\AppData\Local\Temp\WERE0E4.tmp.appcompat.txt
File[5].CabName=memory.hdmp
File[5].Path=memory.hdmp
File[5].Flags=807403520
File[5].Type=3
FriendlyEventName=Stopped working
ConsentKey=BEX64
AppName=beacon.exe
AppPath=C:\users\admin\desktop\beacon.exe
NsPartner=windows
NsGroup=windows8
ApplicationIdentity=4BF1671FF742F6395B0F651AFD44CC12
MetadataHash=1107672753
```

Windows Error Reporting (WER) includes A CSV file **`(WER0A2.tmp.csv)`** that contains additional data about processes running at the time of a system error or crash. Here is a snippet of how the content looks like:

```
ImageName,UniqueProcessId,NumberOfThreads,WorkingSetPrivateSize,HardFaultCount,NumberOfThreadsHighWatermark,CycleTime,CreateTime,UserTime,KernelTime,BasePriority,PeakVirtualSize,VirtualSize,PageFaultCount,WorkingSetSize,PeakWorkingSetSize,QuotaPeakPagedPoolUsage,QuotaPagedPoolUsage,QuotaPeakNonPagedPoolUsage,QuotaNonPagedPoolUsage,PagefileUsage,PeakPagefileUsage,PrivatePageCount,ReadOperationCount,WriteOperationCount,OtherOperationCount,ReadTransferCount,WriteTransferCount,OtherTransferCount,HandleCount,InheritedFromUniqueProcessId,SessionId,Services
"[Idle]",0,5,8192,0,0,170704686832193,133496275036159830,0,527824687500,0,8192,8192,9,8192,12288,0,0,272,272,61440,61440,61440,0,0,0,0,0,0,0,0,0,
"System",4,209,20480,368,225,1105400971390,133496275036159830,0,1716562500,8,32894976,4837376,7981,708608,3858432,0,0,272,272,204800,258048,204800,137,1941,479918,194968,133187147,50821531,3752,0,0,
"Registry",104,4,8286208,2720,11,4293250409,133496275025567068,0,10156250,8,276135936,117932032,117767,33390592,185090048,543552,234560,22472,11696,9449472,29765632,9449472,4,2771,545,16384,65654784,3755,0,4,0,
"smss.exe",468,2,110592,163,4,376979430,133496275036185975,0,1093750,11,2203369410560,2203360768000,1246,962560,2179072,23912,13208,8464,3496,1110016,1191936,1110016,22,8,471,16804,148,21580,56,4,0,
"csrss.exe",576,12,581632,247,14,4580481005,133496275061501495,1562500,10156250,13,2203417059328,2203416072192,4096,4456448,5484544,248072,240296,24584,23312,2002944,2150400,2002944,322,0,9097,249305,0,371948,553,564,0,
"wininit.exe",660,1,8192,254,9,243701774,133496275062067037,156250,468750,13,2203403612160,2203388366848,2585,5357568,7835648,90024,77048,15072,11376,1437696,2113536,1437696,4,1,284,17420,8,4768,162,564,0,
"csrss.exe",672,12,258048,9,17,1164587388,133496275062122989,781250,2343750,13,2203412463616,2203406696448,1775,3907584,5296128,124056,114552,14224,12616,1818624,2154496,1818624,55,0,1283,36202,0,7292,174,652,1,
"winlogon.exe",760,5,446464,399,9,492370537,133496275062381874,468750,1250000,13,2203428392960,2203422973952,7940,18345984,21671936,141248,138984,14256,12352,2633728,3833856,2633728,3,0,279,213868,0,199140,231,652,1,
"services.exe",804,13,3432448,903,39,797834815404,133496275062663765,373281250,1929218750,9,2203424288768,2203414253568,1085226,8880128,14491648,166760,154600,21488,14520,6307840,8404992,6307840,3,4437,4524626,8308,3998256,72049700,692,660,0,
"lsass.exe",828,10,4681728,735,15,22790016395,133496275062979427,36562500,27968750,9,2203419205632,2203418140672,16883,29433856,34000896,139864,139608,29112,25800,8015872,8486912,8015872,216,317,4647,34082,49873,1639232,1417,660,0,KeyIso|SamSs|VaultSvc
"svchost.exe",940,17,7258112,657,33,50185993803,133496275064266237,31875000,99218750,8,2203488657408,2203468070912,40892,44568576,50683904,513696,503424,30176,24808,10870784,11587584,10870784,80,1,73207,327680,65536,1407588,1480,804,0,BrokerInfrastructure|DcomLaunch|PlugPlay|Power|SystemEventsBroker
"fontdrvhost.exe",968,5,24576,45,7,100820860,133496275064408614,468750,312500,8,2203390963712,2203388866560,1858,4640768,5525504,35648,35648,6672,5704,1282048,1355776,1282048,0,0,16,0,0,448,32,760,1,
"fontdrvhost.exe",976,5,90112,51,7,138459095,133496275064408759,0,312500,8,2203391426560,2203389329408,1907,4739072,5615616,36552,36552,6808,5840,1347584,1421312,1347584,0,0,22,0,0,600,32,660,0,
"svchost.exe",540,10,7114752,66,27,78920762929,133496275065037389,116250000,68437500,8,2203461308416,2203425345536,57834,30052352,33976320,174536,174048,32464,22672,8822784,9924608,8822784,20,0,5210,81920,0,288006,1246,804,0,RpcEptMapper|RpcSs
```

| Field                           | Description                                                         |
|---------------------------------|---------------------------------------------------------------------|
| ImageName                       | The name of the executable image (process)                          |
| UniqueProcessId                 | A unique identifier for the process                                 |
| NumberOfThreads                 | The number of threads that the process is running                   |
| WorkingSetPrivateSize           | The size of the working set that is private to the process          |
| HardFaultCount                  | The number of hard faults (page faults that require disk access)    |
| NumberOfThreadsHighWatermark    | The maximum number of threads that were active at any one time      |
| CycleTime                       | The processor cycle time used by the process                        |
| CreateTime                      | The creation time of the process                                    |
| UserTime                        | The amount of time that the process has executed in user mode       |
| KernelTime                      | The amount of time that the process has executed in kernel mode     |
| BasePriority                    | The base priority of the process                                    |
| PeakVirtualSize                 | The peak virtual memory size                                        |
| VirtualSize                     | The current size of the virtual memory that the process is using    |
| PageFaultCount                  | The number of page faults                                           |
| WorkingSetSize                  | The current size of the working set of the process                  |
| PeakWorkingSetSize              | The peak working set size                                           |
| QuotaPeakPagedPoolUsage         | The peak paged pool usage quota for the process                     |
| QuotaPagedPoolUsage             | The current paged pool usage quota                                  |
| QuotaPeakNonPagedPoolUsage      | The peak non-paged pool usage quota                                 |
| QuotaNonPagedPoolUsage          | The current non-paged pool usage quota |
| PagefileUsage                   | The current pagefile usage |
| PeakPagefileUsage               | The peak pagefile usage |
| PrivatePageCount                | The number of private pages used by the process |
| ReadOperationCount              | Count of read I/O operations |
| WriteOperationCount             | Count of write I/O operations |
| OtherOperationCount             | Count of other I/O operations |
| ReadTransferCount               | Amount of data read in operations |
| WriteTransferCount              | Amount of data written in operations |
| OtherTransferCount              | Amount of data transferred in other operations |
| HandleCount                     | The number of handles that the process has open |
| InheritedFromUniqueProcessId    | The process ID of the parent process |
| SessionId                       | The session ID of the process |
| Services                        | The services that are running within the process |

