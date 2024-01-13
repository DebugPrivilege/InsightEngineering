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
