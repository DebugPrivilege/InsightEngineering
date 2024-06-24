# Description

The **`!mex.t`** command is an improved implementation of the **`!thread`** command for both user and kernel mode. It provides information about threads, such as thread context, stack frames, and thread state. The purpose of this section is to be able to learn interpreting the results and how to navigate and examing threads of interest. This is just an example of a write-up that can be used as a reference where I'm demonstrating my thought process. 

# Triaging a thread

We typically begin with the **`!mex.di`** command. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. 

```
0: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 19041 MP (8 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff806`31600000 PsLoadedModuleList = 0xfffff806`3222a270
Debug session time: Fri Oct 15 12:30:10.075 2021 (UTC + 1:00)
System Uptime: 1 days 4:31:51.017
SystemManufacturer = Microsoft Corporation
SystemProductName = Surface Book 2
Processor: Intel(R) Core(TM) i7-8650U CPU @ 1.90GHz
Bugcheck: D1 (8, 2, 0, FFFFF8068A1C75B0)
```

Let's start with the **`!mex.tl -t`** command to discover threads of interest. It examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have running threads, and so on.

```
0: kd> !mex.tl -t
PID            Address          Name                                                    !! Rn Ry Bk Lc IO Er
============== ================ ======================================================= == == == == == == ==
0x0    0n0     fffff80632324a00 Idle                                                     .  8  .  .  .  .  .
0x4    0n4     ffffd80bf5902080 System                                                   8  1  .  .  1  1  .
0x84   0n132   ffffd80bf5aec080 Registry                                                 .  .  .  .  .  .  .
0x248  0n584   ffffd80bfb544040 smss.exe                                                 .  .  .  1  .  .  .
0x384  0n900   ffffd80bff3da100 csrss.exe                                                .  .  .  .  1  .  .
0x3f0  0n1008  ffffd80c02cc8080 wininit.exe                                              .  .  .  .  .  .  .
0x5c   0n92    ffffd80c02cdc100 csrss.exe                                                .  .  .  .  1  .  .
0x8    0n8     ffffd80c02d65340 services.exe                                             .  .  .  .  .  .  .
0x29c  0n668   ffffd80c02dc72c0 LsaIso.exe                                               2  .  .  .  .  .  .
0x408  0n1032  ffffd80c02dca300 lsass.exe                                                .  .  .  .  .  .  .
0x44c  0n1100  ffffd80c02e08080 winlogon.exe                                             .  .  .  .  1  .  .
0x4d8  0n1240  ffffd80c02ebb080 svchost.exe                                              .  .  .  .  .  .  .
0x4f4  0n1268  ffffd80c02f860c0 fontdrvhost.exe                                          .  .  .  .  .  .  .
0x4fc  0n1276  ffffd80c02ebe080 fontdrvhost.exe                                          .  .  .  .  .  .  .
0x51c  0n1308  ffffd80c02f8e080 WUDFHost.exe                                             .  .  .  .  .  .  .
0x588  0n1416  ffffd80c05227080 svchost.exe                                              .  1  .  .  .  .  .
0x5a8  0n1448  ffffd80c0527b0c0 WUDFHost.exe                                             .  .  .  .  .  .  .
0x5dc  0n1500  ffffd80c052a8080 svchost.exe                                              .  .  .  .  .  .  .
0x670  0n1648  ffffd80c05330080 svchost.exe                                              .  .  .  .  .  .  .
0x684  0n1668  ffffd80c0538b280 svchost.exe                                              .  .  .  .  .  .  .
0x6a8  0n1704  ffffd80c0533e080 svchost.exe                                              .  1  .  .  .  .  .
0x6b8  0n1720  ffffd80c05392080 svchost.exe                                              .  .  .  .  .  .  .
0x6e8  0n1768  ffffd80c0534c080 svchost.exe                                              .  .  .  .  .  .  .
0x6f0  0n1776  ffffd80c053970c0 svchost.exe                                              .  .  .  .  .  .  .
0x6f8  0n1784  ffffd80c0534a080 svchost.exe                                              .  .  .  .  .  .  .
0x780  0n1920  ffffd80c0541f080 svchost.exe                                              .  .  .  .  .  .  .
0x788  0n1928  ffffd80c05422080 svchost.exe                                              .  .  .  .  .  .  .
0x7a8  0n1960  ffffd80c053bf080 WUDFHost.exe                                             .  .  .  .  .  .  .
0x7e0  0n2016  ffffd80c0542c080 svchost.exe                                              .  .  .  .  .  .  .
0x830  0n2096  ffffd80c05568080 IntelCpHDCPSvc.exe                                       .  .  .  .  .  .  .
0x850  0n2128  ffffd80c05528080 svchost.exe                                              .  .  .  .  .  .  .
0x874  0n2164  ffffd80c05532080 svchost.exe                                              .  .  .  1  .  .  .
0x890  0n2192  ffffd80c05552080 svchost.exe                                              .  .  .  .  .  .  .
0x8ac  0n2220  ffffd80c0555a280 dwm.exe                                                  .  .  .  .  .  .  .
0x964  0n2404  ffffd80c05643080 svchost.exe                                              .  .  .  .  .  .  .
0x994  0n2452  ffffd80c0567c080 svchost.exe                                              .  .  .  .  .  .  .
0x9dc  0n2524  ffffd80c05689080 IntelCpHeciSvc.exe                                       2  .  .  .  .  .  .
0xaa0  0n2720  ffffd80c056f1080 svchost.exe                                              .  .  .  .  .  .  .
0xaa8  0n2728  ffffd80c056f7080 svchost.exe                                              .  .  2  .  1  .  .
0xafc  0n2812  ffffd80c05768080 NVDisplay.Container.exe                                  5  .  .  .  1  .  .
0xb38  0n2872  ffffd80c05749080 svchost.exe                                              .  .  .  .  .  .  .
0xb68  0n2920  ffffd80c0578b0c0 svchost.exe                                              .  3  .  .  .  .  .
0xb9c  0n2972  ffffd80c057dd0c0 svchost.exe                                              .  .  .  .  .  .  .
0xba4  0n2980  ffffd80c05753080 svchost.exe                                              .  .  2  .  .  .  .
0xbbc  0n3004  ffffd80c057f1080 svchost.exe                                              .  .  .  .  .  .  .
0xbc4  0n3012  ffffd80c057f2080 svchost.exe                                              .  .  .  .  .  .  .
0xbcc  0n3020  ffffd80c057f5080 svchost.exe                                              .  .  .  .  .  .  .
0x7ec  0n2028  ffffd80c058700c0 svchost.exe                                              .  .  .  .  .  .  .
0xc50  0n3152  ffffd80c058bf040 MemCompression                                           .  .  .  .  .  .  .
0xca8  0n3240  ffffd80c0597b080 svchost.exe                                              .  .  .  .  .  .  .
0xcf4  0n3316  ffffd80c059c0080 svchost.exe                                              .  .  .  .  .  .  .
0xd20  0n3360  ffffd80c059f7080 svchost.exe                                              .  .  .  .  .  .  .
0xdb8  0n3512  ffffd80c05a62080 svchost.exe                                              .  .  .  .  .  .  .
0xdc4  0n3524  ffffd80c05a64080 svchost.exe                                              .  .  .  .  .  .  .
0xe3c  0n3644  ffffd80c05b85080 svchost.exe                                              .  .  .  .  .  .  .
0xe70  0n3696  ffffd80c05b660c0 svchost.exe                                              .  .  .  .  .  .  .
0xea4  0n3748  ffffd80c05b75080 svchost.exe                                              .  .  .  .  .  .  .
0xf08  0n3848  ffffd80c05bad080 svchost.exe                                              .  .  .  .  .  .  .
0xf50  0n3920  ffffd80c05bea080 svchost.exe                                              .  .  .  .  2  .  .
0xf98  0n3992  ffffd80c05bf3080 svchost.exe                                              .  .  .  .  .  .  .
0xfd0  0n4048  ffffd80c05c43080 svchost.exe                                              .  .  .  .  .  .  .
0xff4  0n4084  ffffd80c05ce3080 svchost.exe                                              .  .  .  .  .  .  .
0x570  0n1392  ffffd80c05ce70c0 svchost.exe                                              .  .  .  .  .  .  .
0x1020 0n4128  ffffd80c05d64080 svchost.exe                                              .  .  1  .  .  .  .
0x1050 0n4176  ffffd80c05d69080 svchost.exe                                              .  .  1  .  .  .  .
0x1058 0n4184  ffffd80c05d6f080 svchost.exe                                              .  .  .  .  .  .  .
0x10d8 0n4312  ffffd80c05d9b080 svchost.exe                                              .  .  .  .  .  .  .
0x1138 0n4408  ffffd80c05e490c0 spoolsv.exe                                              6  .  .  .  .  .  .
0x1194 0n4500  ffffd80c05e6c080 svchost.exe                                              .  .  .  .  .  .  .
0x121c 0n4636  ffffd80c05ef9080 svchost.exe                                              .  .  .  .  .  .  .
0x1298 0n4760  ffffd80c05f9d0c0 AdobeUpdateService.exe*32                                .  .  .  .  .  .  .
0x12ac 0n4780  ffffd80c05f97080 AGMService.exe*32                                        2  .  .  1  .  .  .
0x12b8 0n4792  ffffd80c05f99080 AGSService.exe*32                                        7  .  .  .  .  .  .
0x12d0 0n4816  ffffd80c05fa0080 svchost.exe                                              .  .  .  .  .  .  .
0x12dc 0n4828  ffffd80c05fa4080 svchost.exe                                              .  .  .  .  .  .  .
0x12e4 0n4836  ffffd80c05fa2080 svchost.exe                                              .  .  .  .  .  .  .
0x12ec 0n4844  ffffd80c05fa5080 vmcompute.exe                                            2  .  .  .  .  .  .
0x12f8 0n4856  ffffd80c05fa8080 svchost.exe                                              .  .  .  .  .  .  .
0x1328 0n4904  ffffd80c05fc5080 esif_uf.exe                                              .  .  .  .  .  .  .
0x1338 0n4920  ffffd80c060020c0 escsvc64.exe                                             .  .  .  .  .  .  .
0x13f0 0n5104  ffffd80c06032080 svchost.exe                                              .  .  1  1  .  .  .
0x13f8 0n5112  ffffd80c060340c0 MSOIDSVC.EXE                                             8  .  .  .  .  .  .
0xdfc  0n3580  ffffd80c0603a080 svchost.exe                                              .  .  .  .  .  .  .
0x1118 0n4376  ffffd80c0602b080 RtkAudUService64.exe                                     .  .  .  .  .  .  .
0xf3c  0n3900  ffffd80c06043080 ss_conn_service.exe*32                                   3  .  .  .  .  .  .
0xee8  0n3816  ffffd80c06092080 svchost.exe                                              .  .  .  .  .  .  .
0x140c 0n5132  ffffd80c060e1080 ss_conn_service2.exe*32                                  3  .  .  .  .  .  .
0x143c 0n5180  ffffd80c060f2080 MsSense.exe                                             11  .  .  .  .  .  .
0x1464 0n5220  ffffd80c06159080 SurfaceColorService.exe                                  .  .  .  .  .  .  .
0x1470 0n5232  ffffd80c06155080 SurfaceDtxService.exe                                    .  .  .  .  .  .  .
0x1478 0n5240  ffffd80c060ee080 SurfaceUsbHubFwUpdateService.exe                         .  .  .  .  .  .  .
0x1480 0n5248  ffffd80c06157080 SurfaceService.exe                                       3  .  .  .  .  .  .
0x14b8 0n5304  ffffd80c061c5080 svchost.exe                                              .  .  .  .  .  .  .
0x14ec 0n5356  ffffd80c061ce080 MsMpEng.exe                                             44  .  .  .  .  .  .
0x1584 0n5508  ffffd80c062e3080 svchost.exe                                              .  .  .  .  .  .  .
0x15f8 0n5624  ffffd80c06350080 svchost.exe                                              .  .  .  .  .  .  .
0x160c 0n5644  ffffd80c063540c0 svchost.exe                                              .  .  .  .  .  .  .
0x1660 0n5728  ffffd80c0639a080 svchost.exe                                              .  .  .  .  .  .  .
0x16c4 0n5828  ffffd80c063c3080 MSOIDSVCM.EXE                                            1  .  .  .  .  .  .
0x17c0 0n6080  ffffd80c0652f080 unsecapp.exe                                             2  .  .  .  .  .  .
0x1944 0n6468  ffffd80c066f0080 svchost.exe                                              .  .  1  .  .  .  .
0x19c8 0n6600  ffffd80c0686d080 svchost.exe                                              .  .  .  .  1  .  .
0x19d0 0n6608  ffffd80c06870080 svchost.exe                                              .  .  .  .  .  .  .
0x1b2c 0n6956  ffffd80c06990080 svchost.exe                                              .  .  .  .  .  .  .
0x1bf8 0n7160  ffffd80c05d7f080 WUDFHost.exe                                             .  .  .  .  .  .  .
0x11c8 0n4552  ffffd80c06aaf080 svchost.exe                                              .  .  .  .  .  .  .
0x14c0 0n5312  ffffd80c06b22080 dllhost.exe                                              .  .  .  .  .  .  .
0x1dc4 0n7620  ffffd80c0706b080 OfficeClickToRun.exe                                    12  .  .  .  .  .  .
0x1c40 0n7232  ffffd80c075e82c0 svchost.exe                                              .  .  1  .  .  .  .
0x1b38 0n6968  ffffd80bfe8ee080 NisSrv.exe                                              19  .  .  .  .  .  .
0x2048 0n8264  ffffd80c07b430c0 svchost.exe                                              .  .  .  .  .  .  .
0x2148 0n8520  ffffd80c07beb0c0 svchost.exe                                              .  .  .  .  .  .  .
0x21f8 0n8696  ffffd80c07cb1080 SgrmBroker.exe                                           4  .  .  .  .  .  .
0x22fc 0n8956  ffffd80c07e51080 svchost.exe                                              .  .  .  .  .  .  .
0x234c 0n9036  ffffd80c07e85080 svchost.exe                                              .  .  .  .  .  .  .
0x2370 0n9072  ffffd80c07ea0080 svchost.exe                                              .  .  .  .  .  .  .
0x23bc 0n9148  ffffd80c07fb5080 svchost.exe                                              .  .  .  .  2  .  .
0x1874 0n6260  ffffd80c0552f080 AppVShNotify.exe                                         .  .  .  .  .  .  .
0x2714 0n10004 ffffd80bfe3a3080 NVDisplay.Container.exe                                  7  .  .  .  .  .  .
0x271c 0n10012 ffffd80c0946f080 svchost.exe                                              .  .  .  .  .  .  .
0x184c 0n6220  ffffd80c09cd7080 svchost.exe                                              .  .  .  .  .  .  .
0x2990 0n10640 ffffd80c09ff1080 dptf_helper.exe                                          .  .  .  .  .  .  .
0x2a60 0n10848 ffffd80c0a8d60c0 sihost.exe                                               .  .  .  .  .  .  .
0x2a8c 0n10892 ffffd80c0a8f2080 svchost.exe                                              .  .  2  .  .  .  .
0x2a98 0n10904 ffffd80c0a8ed080 svchost.exe                                              .  .  .  .  .  .  .
0x2b20 0n11040 ffffd80c07ab4080 svchost.exe                                              .  .  .  .  .  .  .
0x2b78 0n11128 ffffd80c0aaea080 svchost.exe                                              .  .  1  .  .  .  .
0x2bb4 0n11188 ffffd80c0aac7080 SurfaceColorTracker.exe                                  .  .  .  .  .  .  .
0x2844 0n10308 ffffd80c0a7e4080 itype.exe                                                8  .  .  .  .  .  .
0x2874 0n10356 ffffd80c0ab8b080 taskhostw.exe                                            .  .  .  .  .  .  .
0x2084 0n8324  ffffd80c0ab92080 ipoint.exe                                               4  .  .  .  .  .  .
0x2d00 0n11520 ffffd80c0a8da080 svchost.exe                                              .  .  .  .  .  .  .
0x2d8c 0n11660 ffffd80c0ad74140 svchost.exe                                              .  .  .  .  .  .  .
0x2dec 0n11756 ffffd80c0ad96080 explorer.exe                                             6  .  1  1  .  .  .
0xac0  0n2752  ffffd80c0af63340 svchost.exe                                              .  .  .  .  .  .  .
0x2934 0n10548 ffffd80c0b0dd080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x3040 0n12352 ffffd80c0b1860c0 MKCHelper.exe                                            .  .  .  .  .  .  .
0x28b8 0n10424 ffffd80c0aef9080 WmiPrvSE.exe                                             3  .  .  .  .  .  .
0x122c 0n4652  ffffd80c0b1c0080 CcmExec.exe                                             20  .  .  .  1  .  .
0x1f4c 0n8012  ffffd80c0a0cc080 WmiPrvSE.exe                                             3  .  .  .  .  .  .
0x1090 0n4240  ffffd80bfdb0d080 svchost.exe                                              .  .  .  .  .  .  .
0xe00  0n3584  ffffd80bfbbb8080 StartMenuExperienceHost.exe                              2  .  .  .  .  .  .
0x2cd4 0n11476 ffffd80c0b3920c0 RuntimeBroker.exe                                        .  .  .  .  1  .  .
0x341c 0n13340 ffffd80bfd24a080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x3730 0n14128 ffffd80bfd129240 ctfmon.exe                                               2  .  .  .  .  .  .
0x3774 0n14196 ffffd80c09141080 TabTip.exe                                               7  .  .  .  .  .  .
0x3518 0n13592 ffffd80c06b97080 SettingSyncHost.exe                                      9  .  .  .  .  .  .
0x2f4c 0n12108 ffffd80bfd9982c0 TextInputHost.exe                                        1  .  .  .  .  .  .
0x2888 0n10376 ffffd80bfe4f2080 svchost.exe                                              .  .  .  .  .  .  .
0x3768 0n14184 ffffd80bfd105080 svchost.exe                                              .  .  1  .  .  .  .
0x3788 0n14216 ffffd80c1c19b0c0 OUTLOOK.EXE                                             21  .  .  .  .  .  .
0x1488 0n5256  ffffd80c0bffa080 SecurityHealthSystray.exe                                3  .  .  .  .  .  .
0xe98  0n3736  ffffd80c0b472080 SecurityHealthService.exe                                4  .  .  .  .  .  .
0x334c 0n13132 ffffd80bfd132080 SurfaceDTX.exe                                           5  .  .  .  .  .  .
0xe58  0n3672  ffffd80c1c1f8340 RtkAudUService64.exe                                     .  .  .  .  .  .  .
0x3ae0 0n15072 ffffd80c1c1dc080 Greenshot.exe                                            3  .  .  .  .  .  .
0x389c 0n14492 ffffd80c1cc97080 WmiPrvSE.exe                                             4  .  .  .  .  .  .
0xeec  0n3820  ffffd80c1c18a0c0 EPPCCMON.EXE*32                                          2  .  .  .  .  .  .
0x3988 0n14728 ffffd80c1c5bb080 svchost.exe                                              .  .  .  .  .  .  .
0xb4c  0n2892  ffffd80c0552e300 OneDrive.exe                                            14  .  .  .  .  .  .
0x397c 0n14716 ffffd80c1c5b3080 express.exe*32                                          10  .  .  .  .  .  .
0x36d4 0n14036 ffffd80c0b0e7080 OneDrive.exe                                            13  .  .  .  .  .  .
0x1070 0n4208  ffffd80c0bd5c080 ciscowebexstart.exe*32                                   1  .  .  .  .  .  .
0x36cc 0n14028 ffffd80c1c8be080 atmgr.exe*32                                             7  .  .  .  .  .  .
0x3ed0 0n16080 ffffd80bfd1ea080 ONENOTEM.EXE                                             .  .  .  .  .  .  .
0x3c80 0n15488 ffffd80c1f2f8080 notepad.exe                                              3  .  .  .  .  .  .
0x38d8 0n14552 ffffd80c0b4d6080 Spotify.exe*32                                          21  .  .  .  .  .  .
0x3d04 0n15620 ffffd80c0b4d4080 EEventManager.exe*32                                     5  .  .  .  .  .  .
0x2f7c 0n12156 ffffd80c1f648080 dllhost.exe                                              4  .  .  .  .  .  .
0x41b8 0n16824 ffffd80c1f7d00c0 jabra-direct.exe*32                                     15  .  .  .  .  .  .
0x422c 0n16940 ffffd80c1f7e10c0 Spotify.exe*32                                           1  .  .  .  .  .  .
0x4278 0n17016 ffffd80c1f8540c0 msedge.exe                                              10  .  .  .  .  .  .
0x42e0 0n17120 ffffd80c1f8930c0 msedge.exe                                               2  .  .  .  .  .  .
0x43f0 0n17392 ffffd80c1f8d1080 svchost.exe                                              .  .  .  .  .  .  .
0x23b4 0n9140  ffffd80bfb842080 svchost.exe                                              .  .  .  .  .  .  .
0x17b8 0n6072  ffffd80c1f8d0080 Spotify.exe*32                                           8  .  .  .  .  .  .
0x43b0 0n17328 ffffd80c1f972080 Spotify.exe*32                                           2  .  .  .  .  .  .
0x3ef4 0n16116 ffffd80bfe267080 Spotify.exe*32                                           8  .  .  .  .  .  .
0x3114 0n12564 ffffd80c1fae30c0 WmiPrvSE.exe                                             3  .  .  .  .  .  .
0x411c 0n16668 ffffd80c1c0dd080 Spotify.exe*32                                           2  .  .  .  .  .  .
0x456c 0n17772 ffffd80c09eef080 jabra-direct.exe*32                                      8  .  .  .  .  .  .
0x45d0 0n17872 ffffd80bfe48c080 jabra-direct.exe*32                                     10  .  .  .  .  .  .
0x14d8 0n5336  ffffd80c09c4f080 msedge.exe                                               8  .  .  .  .  .  .
0x2524 0n9508  ffffd80c09c4e080 msedge.exe                                               8  .  .  .  5  .  .
0x1514 0n5396  ffffd80c09303080 svchost.exe                                              .  .  .  .  .  .  .
0x24bc 0n9404  ffffd80c2016e240 Teams.exe                                               17  .  .  .  1  .  .
0x4638 0n17976 ffffd80c1f6ec080 jabra-direct.exe*32                                      6  .  .  .  .  .  .
0x4720 0n18208 ffffd80c1f7a3240 svchost.exe                                              .  .  .  .  .  .  .
0x1d38 0n7480  ffffd80c1f7b3080 Microsoft.Management.Services.IntuneWindowsAgent.exe*32  7  .  .  .  .  .  .
0x4584 0n17796 ffffd80c218570c0 SoftphoneIntegrations.exe*32                            16  .  .  .  .  .  .
0x191c 0n6428  ffffd80c2183b0c0 conhost.exe                                              .  .  .  .  .  .  .
0x3dd4 0n15828 ffffd80c21826080 Teams.exe                                                8  .  .  .  .  .  .
0x3f1c 0n16156 ffffd80c1f92e080 CefSharp.BrowserSubprocess.exe*32                        9  .  .  1  .  .  .
0x149c 0n5276  ffffd80c1fed3080 Teams.exe                                               10  .  .  .  2  .  .
0x4874 0n18548 ffffd80c1feaa080 Teams.exe                                                4  .  .  .  .  .  .
0x488c 0n18572 ffffd80c09be0080 msedge.exe                                               1  .  .  .  .  .  .
0x4aa4 0n19108 ffffd80c213860c0 WmiPrvSE.exe                                             2  .  .  .  .  .  .
0x4b94 0n19348 ffffd80c0ba63080 msedge.exe                                               2  .  .  .  .  .  .
0x658  0n1624  ffffd80c1feb0080 CefSharp.BrowserSubprocess.exe*32                        6  .  .  1  .  .  .
0x34a4 0n13476 ffffd80c1f3a7080 Teams.exe                                                4  .  .  .  .  .  .
0x4224 0n16932 ffffd80c2153a080 Teams.exe                                                6  .  .  .  .  .  .
0x45f8 0n17912 ffffd80c21653080 msedge.exe                                               2  .  .  .  .  .  .
0x4eb4 0n20148 ffffd80c211230c0 msedge.exe                                               4  .  .  .  .  .  .
0x4f54 0n20308 ffffd80c2114f080 ShellExperienceHost.exe                                  3  .  .  .  .  .  .
0x4ca4 0n19620 ffffd80c211f00c0 msedge.exe                                               3  .  .  .  .  .  .
0x4828 0n18472 ffffd80c21106080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x3ecc 0n16076 ffffd80c211f1080 Teams.exe                                                4  .  .  .  .  .  .
0x4c90 0n19600 ffffd80c21246080 svchost.exe                                              .  .  .  .  .  .  .
0x20f4 0n8436  ffffd80c2102f080 msedge.exe                                               2  .  .  .  .  .  .
0x4e6c 0n20076 ffffd80c0744e1c0 MBAMAgent.exe                                            3  .  .  .  .  .  .
0xa88  0n2696  ffffd80c06016080 Teams.exe                                               19  .  .  .  .  .  .
0x420c 0n16908 ffffd80c0928b080 uhssvc.exe                                               7  .  .  .  .  .  .
0x183c 0n6204  ffffd80c2064c340 SCNotification.exe                                       7  .  .  .  .  .  .
0x2e9c 0n11932 ffffd80c1f9c5340 svchost.exe                                              .  .  .  .  .  .  .
0x2d84 0n11652 ffffd80bfbbce080 svchost.exe                                              .  .  .  .  .  .  .
0x1e30 0n7728  ffffd80c2114e340 AcrobatNotificationClient.exe*32                         1  .  .  .  .  .  .
0x648  0n1608  ffffd80c21959080 FileCoAuth.exe                                           5  .  .  .  .  .  .
0x4608 0n17928 ffffd80c21a03080 RuntimeBroker.exe                                        .  1  .  .  .  .  .
0x49e4 0n18916 ffffd80c1c108080 ApplicationFrameHost.exe                                 4  .  .  .  .  .  .
0x3574 0n13684 ffffd80c1c864080 WinStore.App.exe                                         2  .  .  .  .  .  .
0x2fa0 0n12192 ffffd80c1cd54080 RuntimeBroker.exe                                        .  .  .  .  1  .  .
0x4108 0n16648 ffffd80c05663080 svchost.exe                                              .  .  .  .  .  .  .
0xffc  0n4092  ffffd80c0b798080 Calculator.exe                                           3  .  .  .  .  .  .
0x239c 0n9116  ffffd80c1cfd2080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x1ac0 0n6848  ffffd80c204d6080 AdobeNotificationClient.exe*32                           1  .  .  .  .  .  .
0x3ea4 0n16036 ffffd80c1c04d080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x4bac 0n19372 ffffd80c1cd1c080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x3f9c 0n16284 ffffd80c1f6ea080 svchost.exe                                              .  .  .  .  .  .  .
0x2650 0n9808  ffffd80c05a05080 SystemSettings.exe                                       3  .  .  .  .  .  .
0x2480 0n9344  ffffd80c1fa880c0 msedge.exe                                               2  .  .  .  .  .  .
0x1fec 0n8172  ffffd80c073a1080 rundll32.exe                                             4  .  .  .  .  .  .
0x136c 0n4972  ffffd80c02df6080 Video.UI.exe                                             5  .  .  .  .  .  .
0x145c 0n5212  ffffd80c21055080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x4220 0n16928 ffffd80c22092080 msedge.exe                                               2  .  .  .  .  .  .
0x13a0 0n5024  ffffd80c1e3a50c0 msedge.exe                                               2  .  .  .  .  .  .
0x46bc 0n18108 ffffd80c2120b080 Microsoft.Photos.exe                                     4  .  .  .  .  .  .
0x175c 0n5980  ffffd80c24213340 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x3a38 0n14904 ffffd80c242a7080 Kusto.Explorer.exe*32                                   11  .  .  .  .  .  .
0x3cd0 0n15568 ffffd80c1ed390c0 msoasb.exe                                               5  .  .  .  .  .  .
0x2ae4 0n10980 ffffd80c0b129080 svchost.exe                                              .  .  .  .  .  .  .
0xcb8  0n3256  ffffd80c00cac080 dllhost.exe                                              5  .  .  .  .  .  .
0x4d4c 0n19788 ffffd80c0b657080 msdtc.exe                                                .  .  .  1  .  .  .
0x4100 0n16640 ffffd80c25c54080 msedge.exe                                               3  .  .  .  .  .  .
0x14c8 0n5320  ffffd80c0adc2080 svchost.exe                                              .  .  1  .  1  .  .
0x18ec 0n6380  ffffd80c1c893080 SearchIndexer.exe                                       44  .  .  .  .  .  .
0x3718 0n14104 ffffd80c24634300 armsvc.exe*32                                            .  .  .  .  .  .  .
0x5a88 0n23176 ffffd80bfdab9080 powershell.exe                                           5  .  .  .  .  .  .
0x4d64 0n19812 ffffd80c1ff04080 conhost.exe                                              .  .  .  .  .  .  .
0x5b7c 0n23420 ffffd80c21831080 msedge.exe                                               2  .  .  .  .  .  .
0x668  0n1640  ffffd80c262130c0 msedge.exe                                               4  .  .  .  .  .  .
0x3d7c 0n15740 ffffd80c200ac300 svchost.exe                                              .  .  .  .  .  .  .
0x45a0 0n17824 ffffd80c2457c080 WmiPrvSE.exe                                            10  .  .  .  .  .  .
0x56f0 0n22256 ffffd80c206db080 svchost.exe                                              .  .  .  .  .  .  .
0x5738 0n22328 ffffd80c25fe4080 svchost.exe                                              .  .  .  .  .  .  .
0x5d94 0n23956 ffffd80c1dbf0140 WUDFHost.exe                                             .  .  .  .  .  .  .
0x1204 0n4612  ffffd80c1c9f1080 WUDFHost.exe                                             .  .  .  .  .  .  .
0x5644 0n22084 ffffd80c1e8110c0 YourPhone.exe                                            3  .  .  .  .  .  .
0x63f8 0n25592 ffffd80c25861080 SenseCE.exe                                              1  .  .  .  .  .  .
0xb78  0n2936  ffffd80c09b790c0 SurfaceAudio.exe                                         3  .  .  .  .  .  .
0x45b0 0n17840 ffffd80c217cb0c0 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x6048 0n24648 ffffd80c200350c0 svchost.exe                                              .  .  .  .  .  .  .
0x2a0c 0n10764 ffffd80c28bb00c0 svchost.exe                                              .  .  .  .  .  .  .
0x4d2c 0n19756 ffffd80c1e986080 svchost.exe                                              .  .  .  .  .  .  .
0x5b10 0n23312 ffffd80c07cb5080 dllhost.exe                                              3  .  .  .  .  .  .
0x531c 0n21276 ffffd80c1c01e080 dasHost.exe                                              .  .  .  .  .  .  .
0x125c 0n4700  ffffd80c0b8c3080 GpVpnApp.exe                                             5  .  .  .  .  .  .
0x342c 0n13356 ffffd80c09c130c0 SearchApp.exe                                            .  1  .  .  3  .  .
0x5028 0n20520 ffffd80c2666e0c0 taskhostw.exe                                            4  .  .  .  .  .  .
0x6204 0n25092 ffffd80c0bae80c0 msedge.exe                                               3  .  .  .  .  .  .
0x39e4 0n14820 ffffd80c1cc670c0 RuntimeBroker.exe                                        .  .  .  1  .  .  .
0x55a8 0n21928 ffffd80c24ad90c0 msedge.exe                                               3  .  .  .  .  .  .
0x251c 0n9500  ffffd80c0b0800c0 msedge.exe                                               2  .  .  .  .  .  .
0x4d30 0n19760 ffffd80bfd2490c0 msedge.exe                                               1  .  .  .  .  .  .
0x5660 0n22112 ffffd80c21775080 msedge.exe                                               2  .  .  .  .  .  .
0x494c 0n18764 ffffd80c06bd90c0 msedge.exe                                               2  .  .  .  .  .  .
0x14e0 0n5344  ffffd80c1fbc90c0 msedge.exe                                               3  .  .  .  .  .  .
0x2498 0n9368  ffffd80c21035080 msedge.exe                                               4  .  .  .  .  .  .
0x36a4 0n13988 ffffd80c1d9d5080 msedge.exe                                               2  .  .  .  .  .  .
0x4138 0n16696 ffffd80c24647080 msedge.exe                                               2  .  .  .  .  .  .
0x319c 0n12700 ffffd80bfdbe3080 ONENOTE.EXE                                             11  .  .  .  .  .  .
0x29f4 0n10740 ffffd80c21030080 msedge.exe                                               2  .  .  .  .  .  .
0x32b8 0n12984 ffffd80c21a51080 msedge.exe                                               3  .  .  .  .  .  .
0x472c 0n18220 ffffd80bfd9a8080 msedge.exe                                               2  .  .  .  .  .  .
0xd54  0n3412  ffffd80c0710c2c0 msedge.exe                                               2  .  .  .  .  .  .
0x559c 0n21916 ffffd80c25397080 msedge.exe                                               2  .  .  .  .  .  .
0x5794 0n22420 ffffd80c24e20080 msedge.exe                                               3  .  .  .  .  .  .
0x4c8c 0n19596 ffffd80c0ab7d0c0 msedge.exe                                               3  .  .  .  .  .  .
0x4afc 0n19196 ffffd80c24fbc0c0 LogonUI.exe                                              .  .  .  .  1  .  .
0x33ec 0n13292 ffffd80c09ce6080 taskhostw.exe                                            6  .  .  .  .  .  .
0x2040 0n8256  ffffd80c0b8c60c0 TabTip.exe                                               7  .  .  .  .  .  .
0x3340 0n13120 ffffd80c246af080 msedge.exe                                               2  .  .  .  .  .  .
0x5904 0n22788 ffffd80c0b4d9080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x30c4 0n12484 ffffd80c09c75080 MoUsoCoreWorker.exe                                      8  .  .  .  .  .  .
0x4e90 0n20112 ffffd80bfd4f0080 TrustedInstaller.exe                                     6  .  .  .  .  .  .
0x2d9c 0n11676 ffffd80c23ea1080 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x4494 0n17556 ffffd80c1eac50c0 RuntimeBroker.exe                                        .  .  .  .  .  .  .
0x6388 0n25480 ffffd80c073a9080 TiWorker.exe                                             6  .  .  .  .  .  .
0x321c 0n12828 ffffd80c1e617080 msedge.exe                                              15  .  .  .  .  .  .
0x3c94 0n15508 ffffd80c1ebda080 Spotify.exe*32                                           2  .  .  .  .  .  .
0x4590 0n17808 ffffd80c1c1c7080 identity_helper.exe                                      7  .  .  .  .  .  .
0x45e4 0n17892 ffffd80c1c554340 WmiPrvSE.exe                                             5  .  .  .  .  .  .
0x1724 0n5924  ffffd80c1cc6d080 svchost.exe                                              .  .  .  .  .  .  .
0x1dac 0n7596  ffffd80c2426b080 Teams.exe                                                3  .  .  .  .  .  .
0x2654 0n9812  ffffd80c1f85c080 msedge.exe                                               1  .  .  .  .  .  .
0x53c0 0n21440 ffffd80c21bce080 WmiApSrv.exe                                             5  .  .  .  .  .  .
0x54bc 0n21692 ffffd80c256b1080 dllhost.exe                                             13  .  .  .  .  .  .
0x6370 0n25456 ffffd80c1ccaf0c0 svchost.exe                                              .  .  .  .  .  .  .
0x5074 0n20596 ffffd80bfd7b3180 svchost.exe                                              .  .  .  .  .  .  .
============== ================ ======================================================= == == == == == == ==
PID            Address          Name                                                    !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 29 [zombie report]
```

If we start to categorize the threads, we can identify several in a **ready** state, as well as some that are **running**. These threads are of particular interest and should be examined further.

```
0x1020 0n4128  ffffd80c05d64080 svchost.exe                                              .  .  1  .  .  .  .
0x1050 0n4176  ffffd80c05d69080 svchost.exe                                              .  .  1  .  .  .  .
0x13f0 0n5104  ffffd80c06032080 svchost.exe                                              .  .  1  1  .  .  .
0x1944 0n6468  ffffd80c066f0080 svchost.exe                                              .  .  1  .  .  .  .
0x1c40 0n7232  ffffd80c075e82c0 svchost.exe                                              .  .  1  .  .  .  .
0x2b78 0n11128 ffffd80c0aaea080 svchost.exe                                              .  .  1  .  .  .  .
0x2dec 0n11756 ffffd80c0ad96080 explorer.exe                                             6  .  1  1  .  .  .
0x3768 0n14184 ffffd80bfd105080 svchost.exe                                              .  .  1  .  .  .  .
0x14c8 0n5320  ffffd80c0adc2080 svchost.exe                                              .  .  1  .  1  .  .
0xaa8  0n2728  ffffd80c056f7080 svchost.exe                                              .  .  2  .  1  .  .
0xba4  0n2980  ffffd80c05753080 svchost.exe                                              .  .  2  .  .  .  .
0x2a8c 0n10892 ffffd80c0a8f2080 svchost.exe                                              .  .  2  .  .  .  .
0x4    0n4     ffffd80bf5902080 System                                                   8  1  .  .  1  1  .
0x588  0n1416  ffffd80c05227080 svchost.exe                                              .  1  .  .  .  .  .
0x6a8  0n1704  ffffd80c0533e080 svchost.exe                                              .  1  .  .  .  .  .
0x4608 0n17928 ffffd80c21a03080 RuntimeBroker.exe                                        .  1  .  .  .  .  .
0x342c 0n13356 ffffd80c09c130c0 SearchApp.exe                                            .  1  .  .  3  .  .
0xb68  0n2920  ffffd80c0578b0c0 svchost.exe                                              .  3  .  .  .  .  .
0x0    0n0     fffff80632324a00 Idle                                                     .  8  .  .  .  .  .
============== ================ ======================================================= == == == == == == ==
PID            Address          Name                                                    !! Rn Ry Bk Lc IO Er
```

The first thing we need to understand is the thread **state**. The **state** of a thread will determine how you analyze everything else. If the thread is in a waiting state, you need to know how long it has been waiting and why it is waiting. If it is running, you want to know how long it has been running. However, we don't really need to know why it is running; it's on a CPU and executing.

```
0: kd> !mex.trep
Count State
===== =======
    1 Standby
    8 Running
   14 Ready
4,081 Waiting
===== =======
4,104 
```

A thread in the **Standby** state has been selected by the scheduler to run next on a specific CPU, while a thread in the **Ready** state is prepared to run but has not yet been chosen by the scheduler. These threads are in a queue, waiting for the scheduler to select them for execution.

```
0: kd> !mex.Standby
Process      PID Thread             Id Pri Base Pri Affinity Next CPU Ideal CPU CSwitches User Kernel State   Time Reason
=========== ==== ================ ==== === ======== ======== ======== ========= ========= ==== ====== ======= ==== ===========
svchost.exe 13f0 ffffd80c05b7c040 4a20  15        8        1        0         0       199    0      0 Standby    0 WrPreempted

Count: 1
```

This thread has been on standby for **0s** waiting for **CPU 0**. The Wait Reason is set to **`WrPreempted`** which refers to a thread that was paused on a CPU because a more important thread needed to use that CPU.

```
0: kd> !mex.t ffffd80c05b7c040
Process                        Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason Time State
svchost.exe (ffffd80c06032080) ffffd80c05b7c040 (E|K|W|R|V) 13f0.4a20 00000009176ac000        0          0             199 WrPreempted    0 Standby on processor 0

Irp List:
    IRP              File Driver
    ffffd80c1c3625a0      tunnel

Priority:
    Current Base Decrement ForegroundBoost IO Page
    15      8    0         96              0  5

 # Child-SP         Return           Call Site
 0 fffffe836a45f100 fffff806318140c2 nt!KiSwapContext+0x76
 1 fffffe836a45f240 fffff806318f1c9a nt!KiProcessDeferredReadyList+0x112
 2 fffffe836a45f280 fffff806318e4622 nt!KeSetSystemGroupAffinityThread+0x13a
 3 fffffe836a45f2f0 fffff8063193feb7 nt!KeGenericProcessorCallback+0x10e
 4 fffffe836a45f460 fffff80633fdf97e nt!KeGenericCallDpc+0x27
 5 fffffe836a45f4a0 fffff806343aa82a NETIO!KfdDeleteCalloutEntry+0x2e
 6 fffffe836a45f4d0 fffff806343aaab9 fwpkclnt!FwppCalloutUnregister+0x62
 7 fffffe836a45f510 fffff8068a1cbabc fwpkclnt!FwpsCalloutUnregisterById0+0x49
 8 fffffe836a45f550 fffff8068a1ca2d5 tunnel!TeredoWfpUnregisterCallouts+0x20
 9 fffffe836a45f580 fffff8068a1ce2e3 tunnel!TeredoUnregisterWithWfp+0x41
 a fffffe836a45f5b0 fffff8068a1c5131 tunnel!TunnelUserHaltTunnel+0x113
 b fffffe836a45f5f0 fffff8068a1cbe63 tunnel!TunnelMiniportHaltLWM+0xf9
 c fffffe836a45f650 fffff8063196d0b7 tunnel!TunnelDriverDispatchDevControl+0xc3
 d fffffe836a45f690 fffff80631b02c7b nt!IopfCallDriver+0x53
 e fffffe836a45f6d0 fffff80631a3eda1 nt!IopPerfCallDriver+0xb3
 f fffffe836a45f700 fffff80631c758f8 nt!IofCallDriver+0x1af701
10 fffffe836a45f740 fffff80631c751c5 nt!IopSynchronousServiceTail+0x1a8
11 fffffe836a45f7e0 fffff80631c74bc6 nt!IopXxxControlFile+0x5e5
12 fffffe836a45f920 fffff80631a08bb5 nt!NtDeviceIoControlFile+0x56
13 fffffe836a45f990 00007ffc040ace54 nt!KiSystemServiceCopyEnd+0x25
14 0000000917d7f3b8 0000000000000000 0x7ffc040ace54
```

We know that the thread is on **standby** waiting for **CPU 0** and that the Wait Reason is set to **`WrPreempted`** which refers to a thread that was paused on a CPU because a more important thread needed to use that CPU. The question that we now should ask ourselves is which thread we are talking about?

In this example, the running thread is **`ffffd80c204a6040`** that was using the **CPU 0** - What is this thread doing?

```
7: kd> !pcr 0
KPCR for Processor 0 at fffff8062b56e000:
    Major 1 Minor 1
	NtTib.ExceptionList: fffff80634ca3fb0
	    NtTib.StackBase: fffff80634ca2000
	   NtTib.StackLimit: 0000000000000000
	 NtTib.SubSystemTib: fffff8062b56e000
	      NtTib.Version: 000000002b56e180
	  NtTib.UserPointer: fffff8062b56e870
	      NtTib.SelfTib: 00000025b94d5000

	            SelfPcr: 0000000000000000
	               Prcb: fffff8062b56e180
	               Irql: 0000000000000000
	                IRR: 0000000000000000
	                IDR: 0000000000000000
	      InterruptMode: 0000000000000000
	                IDT: 0000000000000000
	                GDT: 0000000000000000
	                TSS: 0000000000000000

	      CurrentThread: ffffd80c204a6040
	         NextThread: ffffd80c05b7c040
	         IdleThread: fffff80632327a00

	          DpcQueue:
```

The thread in question is part of the **`System`** process and was running on **CPU 0** at the time of the crash. It experienced a kernel mode page fault in the **`tunnel.sys`** driver, which let to the thread crashing.

```
0: kd> !mex.t ffffd80c204a6040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffd80bf5902080) ffffd80c204a6040 (E|K|W|R|V) 4.2fc            0      219ms            9491 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    12      12   0         0               0  5

 # Child-SP         Return           Call Site
 0 fffff80634cb65b8 fffff80631a09169 nt!KeBugCheckEx
 1 fffff80634cb65c0 fffff80631a05469 nt!KiBugCheckDispatch+0x69
 2 fffff80634cb6700 fffff8068a1c75b0 nt!KiPageFault+0x469 --> Trapframe
 3 fffff80634cb6890 fffff8068a1ca615 tunnel!TeredoWfpRemoveHashEntry+0x2c
 4 fffff80634cb68c0 fffff8068a1ca5e3 tunnel!TeredoWfpDereferenceFlowContextImp+0x1d
 5 fffff80634cb68f0 fffff80633fa0577 tunnel!TeredoWfpDdLayerDelete+0x53
 6 fffff80634cb6920 fffff80633fa0037 NETIO!WfpNotifyFlowContextDelete+0x20b
 7 fffff80634cb69a0 fffff806340eb580 NETIO!KfdAleNotifyFlowDeletion+0x1c7
 8 fffff80634cb6a00 fffff806340b929a tcpip!WfpAleFreeRemoteEndpoint+0x30
 9 fffff80634cb6a80 fffff806340b7407 tcpip!WfpAleDecrementWaitRef+0x76
 a fffff80634cb6ab0 fffff806340b7358 tcpip!WfpAleDeleteRemoteEndpointLru+0x9b
 b fffff80634cb6b00 fffff8063189a24e tcpip!LruCleanupDpcRoutine+0x378
 c fffff80634cb6bb0 fffff80631899534 nt!KiExecuteAllDpcs+0x30e
 d fffff80634cb6d20 fffff806319fe1f5 nt!KiRetireDpcList+0x1f4
 e fffff80634cb6fb0 fffff806319fdfe0 nt!KxRetireDpcList+0x5
 f fffffe836ceba200 fffff806319fd6ae nt!KiDispatchInterruptContinue
10 fffffe836ceba230 fffff8063183a23b nt!KiDpcInterrupt+0x2ee
11 fffffe836ceba3c0 fffff806340a4e94 nt!KzLowerIrql+0x1b
12 fffffe836ceba3f0 fffff80634100026 tcpip!IppSendDatagramsCommon+0x4c4
13 fffffe836ceba570 fffff806340ffc16 tcpip!IppProcessMulticastDiscoveryTimeoutEvents+0x3ca
14 fffffe836ceba9b0 fffff80631874825 tcpip!IppMulticastWorkerRoutine+0xa6
15 fffffe836cebaa00 fffff806318b8515 nt!IopProcessWorkItem+0x135
16 fffffe836cebaa70 fffff80631955855 nt!ExpWorkerThread+0x105
17 fffffe836cebab10 fffff806319fe808 nt!PspSystemThreadStartup+0x55
18 fffffe836cebab60 0000000000000000 nt!KiStartSystemThread+0x28
```

The **`.trap`** command is used to display the trap frame that represents the state of the processor when an exception (such as a page fault) occurred. The results look odd, but let's get back to that later.

```
7: kd> .trap fffff80634cb6700
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=0000000000000002 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=fffff8068a1c75b0 rsp=fffff80634cb6890 rbp=ffffd80c250e08b0
 r8=ffffd80c28de9530  r9=ffffd80c28de9530 r10=fffff806318d7910
r11=0000000000000002 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei ng nz na pe nc
tunnel!RemoveEntryList+0x3 [inlined in tunnel!TeredoWfpRemoveHashEntry+0x2c]:
fffff806`8a1c75b0 48395a08        cmp     qword ptr [rdx+8],rbx ds:00000000`00000008=????????????????
```

There are 14 threads that are in the **ready** state, meaning they are prepared to execute but are currently waiting for a CPU to become available. 

```
0: kd> !mex.ready
Process       PID Thread             Id Pri Base Pri Affinity Next CPU Ideal CPU CSwitches    User  Kernel State Time Reason
============ ==== ================ ==== === ======== ======== ======== ========= ========= ======= ======= ===== ==== =======
svchost.exe  1020 ffffd80c06672040 554c   8        8      255        0         0       130    16ms       0 Ready    0 WrQueue
svchost.exe  1050 ffffd80c21504080 54b4   8        8      255        0         0       652    16ms       0 Ready    0 WrQueue
svchost.exe   aa8 ffffd80c2005c2c0 22e0   8        8      255        1         1        63       0    16ms Ready    0 WrQueue
svchost.exe   ba4 ffffd80c1e63d080  f20   8        8      255        1         1       152       0       0 Ready    0 WrQueue
svchost.exe   aa8 ffffd80c1c4c05c0 2efc   8        8      255        2         2      1874    78ms    47ms Ready    0 WrQueue
svchost.exe  1c40 ffffd80c25803080 51f0   8        8      255        3         3      5814    31ms    78ms Ready    0 WrQueue
svchost.exe  2a8c ffffd80c0a3d9040 2f18   8        8      255        2         2     13921   141ms   141ms Ready    0 WrQueue
svchost.exe   ba4 ffffd80bfd39c2c0 57d8   8        8      255        7         7       278       0    16ms Ready    0 WrQueue
svchost.exe  14c8 ffffd80c1c5ce040  90c   8        8      255        7         7       287    16ms       0 Ready    0 WrQueue
svchost.exe  3768 ffffd80c24dea080 6254   8        8      255        4         4       110    16ms    16ms Ready    0 WrQueue
svchost.exe  1944 ffffd80c1f4d94c0 2018   8        8      255        4         4         3       0       0 Ready    0 WrQueue
svchost.exe  2a8c ffffd80bfd467380 2e38   8        8      255        6         6      2170       0       0 Ready    0 WrQueue
explorer.exe 2dec ffffd80bfd4e9080 3584   8        8      255        4         4    195584 12s.047 11s.656 Ready    0 WrQueue
svchost.exe  2b78 ffffd80c02fae080 4f44   8        8      255        5         5        62       0       0 Ready    0 WrQueue

Count: 14
```

When deciding which thread in **`svchost.exe`** to analyze first, it’s generally a good idea to start with the thread that has the highest activity or **context switches** (CSwitches). A context switch occurs when the CPU changes from executing one thread to executing another. This involves saving the state of the current thread and loading the state of the next thread to be executed. A high number of context switches can introduce CPU overhead, reduce cache efficiency, and increase latency, leading to overall performance issues.

This thread has been ready for **0s** on **CPU 2**, but which thread was **running** on **CPU2**?

```
0: kd> !mex.t ffffd80c0a3d9040
Process                        Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason Time State
svchost.exe (ffffd80c0a8f2080) ffffd80c0a3d9040 (E|K|W|R|V) 2a8c.2f18 0000009e6d452000    141ms      141ms           13921 WrQueue        0 Ready

# Child-SP         Return           Call Site
0 fffffe836fdbf300 fffff8063180c800 nt!KiSwapContext+0x76
1 fffffe836fdbf440 fffff8063180bd2f nt!KiSwapThread+0x500
2 fffffe836fdbf4f0 fffff8063180f663 nt!KiCommitThreadWait+0x14f
3 fffffe836fdbf590 fffff8063180f098 nt!KeRemoveQueueEx+0x263
4 fffffe836fdbf630 fffff8063181012e nt!IoRemoveIoCompletion+0x98
5 fffffe836fdbf760 fffff80631a08bb5 nt!NtWaitForWorkViaWorkerFactory+0x38e
6 fffffe836fdbf990 00007ffc040b07c4 nt!KiSystemServiceCopyEnd+0x25
7 0000009e6e07f808 0000000000000000 0x7ffc040b07c4
```

In this example, the thread **`ffffd80c05988080`** was actively **running** on **CPU 2**. We can now start examining this thread to better understand what it was doing.

```
0: kd> !pcr 2
KPCR for Processor 2 at ffff8981b8300000:
    Major 1 Minor 1
	NtTib.ExceptionList: ffff8981b830ffb0
	    NtTib.StackBase: ffff8981b830e000
	   NtTib.StackLimit: 0000000000000000
	 NtTib.SubSystemTib: ffff8981b8300000
	      NtTib.Version: 00000000b8300180
	  NtTib.UserPointer: ffff8981b8300870
	      NtTib.SelfTib: 00000025b94c5000

	            SelfPcr: 0000000000000000
	               Prcb: ffff8981b8300180
	               Irql: 0000000000000000
	                IRR: 0000000000000000
	                IDR: 0000000000000000
	      InterruptMode: 0000000000000000
	                IDT: 0000000000000000
	                GDT: 0000000000000000
	                TSS: 0000000000000000

	      CurrentThread: ffffd80c05988080
	         NextThread: 0000000000000000
	         IdleThread: ffff8981b830b240

	          DpcQueue:
```

The thread **`ffffd80c05988080`** in the **`svchost.exe`** process is actively **running** on **CPU 2**. It is currently spinning and trying to acquire a **NDIS write lock**. This means it is continuously attempting to obtain the lock and cannot proceed until it succeeds. Since the thread is trying to acquire a write lock and is unable to proceed, it indicates that the lock is currently held by another thread, which is causing the current thread to wait.

```
0: kd> !mex.t ffffd80c05988080
Process                        Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason Time State
svchost.exe (ffffd80c0578b0c0) ffffd80c05988080 (E|K|W|R|V) b68.cc8   00000025b94c5000   3s.031    10s.547           11504 Executive      0 Running on processor 2

Irp List:
    IRP              File      Driver
    ffffd80c1f7a07a0 \Endpoint AFD
    ffffd80bfd5d17a0 \Endpoint AFD

Priority:
    Current Base Decrement ForegroundBoost IO Page
    10      8    0         0               0  5

 # Child-SP         Return           Call Site
 0 fffffe836850f0b0 fffff806318d7998 nt!KxWaitForSpinLockAndAcquire+0x2c
 1 fffffe836850f0e0 fffff80633e1432e nt!KeAcquireSpinLockRaiseToDpc+0x88
 2 fffffe836850f110 fffff806340b8afa ndis!NdisAcquireRWLockWrite+0x3e --> Trying to acquire a NDIS RW Lock
 3 fffffe836850f140 fffff806340b7fb9 tcpip!UdpCloseEndpoint+0xb32
 4 fffffe836850f480 fffff806456c5834 tcpip!UdpTlProviderCloseEndpoint+0x9
 5 fffffe836850f4b0 fffff806456a5831 afd!AfdTLCloseEndpoint+0x48
 6 fffffe836850f4f0 fffff806456c2824 afd!AfdCloseTransportEndpoint+0x89
 7 fffffe836850f5d0 fffff806456c2c6c afd!AfdCleanupCore+0x3b8
 8 fffffe836850f6d0 fffff8063196d0b7 afd!AfdDispatch+0xec
 9 fffffe836850f710 fffff80631b02c7b nt!IopfCallDriver+0x53
 a fffffe836850f750 fffff80631a3eda1 nt!IopPerfCallDriver+0xb3
 b fffffe836850f780 fffff80631c75d4a nt!IofCallDriver+0x1af701
 c fffffe836850f7c0 fffff80631bf39cf nt!IopCloseFile+0x17a
 d fffffe836850f850 fffff80631bf79ac nt!ObCloseHandleTableEntry+0x51f
 e fffffe836850f990 fffff80631a08bb5 nt!NtClose+0xec
 f fffffe836850fa00 00007ffc040acf54 nt!KiSystemServiceCopyEnd+0x25
10 00000025b9bff6d8 0000000000000000 0x7ffc040acf54
```

The command **`.frame /r 2`** sets the context to the 2th frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point.

This is a snapshot of the CPU state at the 2th frame of the stack, which is within the **`ndis!NdisAcquireRWLockWrite`** function.

```
0: kd> .frame /r 0n2
02 fffffe83`6850f110 fffff806`340b8afa     ndis!NdisAcquireRWLockWrite+0x3e
rax=0000000000000001 rbx=ffffd80bf90ded60 rcx=ffffd80bf90ded70
rdx=fffffe836850f190 rsi=fffffe836850f190 rdi=ffffd80c05988080
rip=fffff80633e1432e rsp=fffffe836850f110 rbp=fffffe836850f240
 r8=0000000000000000  r9=0000000000000000 r10=fffff806318d7910
r11=fffffe836850f0e0 r12=0000000000000180 r13=0000000000000007
r14=ffffd80c1c746e30 r15=0000000000000003
iopl=0         nv up ei pl nz na pe nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040202
ndis!NdisAcquireRWLockWrite+0x3e:
fffff806`33e1432e 8806            mov     byte ptr [rsi],al ds:002b:fffffe83`6850f190=00
```

The **`RBX`** register in the context of the **`NdisAcquireRWLockWrite`** function holds the address of the read-write lock data structure. The lock is currently held in exclusive mode, which means it has been acquired for writing. In this mode, no other thread can acquire the lock for reading or writing until it is released.

```
0: kd> !ndiskd.ndisrwlock ffffd80bf90ded60


NDIS READ-WRITE LOCK

    Allocated by       [NDIS generic object]
    Exclusive access   Acquired by thread ffffd80c204a6040 [Current]
```

The thread **`ffffd80c204a6040`** is currently holding the NDIS read-write lock and is involved in network cleanup operations. However, it is also in the process of crashing, as indicated by the call to **`nt!KeBugCheckEx`**. 

```
0: kd> !mex.t ffffd80c204a6040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffd80bf5902080) ffffd80c204a6040 (E|K|W|R|V) 4.2fc            0      219ms            9491 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    12      12   0         0               0  5

 # Child-SP         Return           Call Site
 0 fffff80634cb65b8 fffff80631a09169 nt!KeBugCheckEx
 1 fffff80634cb65c0 fffff80631a05469 nt!KiBugCheckDispatch+0x69
 2 fffff80634cb6700 fffff8068a1c75b0 nt!KiPageFault+0x469
 3 fffff80634cb6890 fffff8068a1ca615 tunnel!TeredoWfpRemoveHashEntry+0x2c
 4 fffff80634cb68c0 fffff8068a1ca5e3 tunnel!TeredoWfpDereferenceFlowContextImp+0x1d
 5 fffff80634cb68f0 fffff80633fa0577 tunnel!TeredoWfpDdLayerDelete+0x53
 6 fffff80634cb6920 fffff80633fa0037 NETIO!WfpNotifyFlowContextDelete+0x20b
 7 fffff80634cb69a0 fffff806340eb580 NETIO!KfdAleNotifyFlowDeletion+0x1c7
 8 fffff80634cb6a00 fffff806340b929a tcpip!WfpAleFreeRemoteEndpoint+0x30
 9 fffff80634cb6a80 fffff806340b7407 tcpip!WfpAleDecrementWaitRef+0x76
 a fffff80634cb6ab0 fffff806340b7358 tcpip!WfpAleDeleteRemoteEndpointLru+0x9b
 b fffff80634cb6b00 fffff8063189a24e tcpip!LruCleanupDpcRoutine+0x378
 c fffff80634cb6bb0 fffff80631899534 nt!KiExecuteAllDpcs+0x30e
 d fffff80634cb6d20 fffff806319fe1f5 nt!KiRetireDpcList+0x1f4
 e fffff80634cb6fb0 fffff806319fdfe0 nt!KxRetireDpcList+0x5
 f fffffe836ceba200 fffff806319fd6ae nt!KiDispatchInterruptContinue
10 fffffe836ceba230 fffff8063183a23b nt!KiDpcInterrupt+0x2ee
11 fffffe836ceba3c0 fffff806340a4e94 nt!KzLowerIrql+0x1b
12 fffffe836ceba3f0 fffff80634100026 tcpip!IppSendDatagramsCommon+0x4c4
13 fffffe836ceba570 fffff806340ffc16 tcpip!IppProcessMulticastDiscoveryTimeoutEvents+0x3ca
14 fffffe836ceba9b0 fffff80631874825 tcpip!IppMulticastWorkerRoutine+0xa6
15 fffffe836cebaa00 fffff806318b8515 nt!IopProcessWorkItem+0x135
16 fffffe836cebaa70 fffff80631955855 nt!ExpWorkerThread+0x105
17 fffffe836cebab10 fffff806319fe808 nt!PspSystemThreadStartup+0x55
18 fffffe836cebab60 0000000000000000 nt!KiStartSystemThread+0x28
```

The crash of the thread holding the NDIS read-write lock leads to a situation where the lock cannot be released. This can lead to a deadlock for other threads waiting to acquire the same lock, such as **`ffffd80c05988080`**. 

To finish it off, we will run the **`!analyze -v`** command.

```
0: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: 0000000000000008, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000000, value 0 = read operation, 1 = write operation
Arg4: fffff8068a1c75b0, address which referenced memory

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 5437

    Key  : Analysis.Elapsed.mSec
    Value: 5535

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 6

    Key  : Analysis.IO.Write.Mb
    Value: 13

    Key  : Analysis.Init.CPU.mSec
    Value: 43687

    Key  : Analysis.Init.Elapsed.mSec
    Value: 16632329

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 767

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0xd1

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0xd1

    Key  : Bugcheck.Code.TargetModel
    Value: 0xd1

    Key  : Failure.Bucket
    Value: AV_tunnel!TeredoWfpRemoveHashEntry

    Key  : Failure.Hash
    Value: {ed81a04d-9dd2-aa51-bc2a-d6f45f90b7a5}

    Key  : Hypervisor.Enlightenments.Value
    Value: 68669340

    Key  : Hypervisor.Enlightenments.ValueHex
    Value: 417cf9c

    Key  : Hypervisor.Flags.AnyHypervisorPresent
    Value: 1

    Key  : Hypervisor.Flags.ApicEnlightened
    Value: 1

    Key  : Hypervisor.Flags.ApicVirtualizationAvailable
    Value: 0

    Key  : Hypervisor.Flags.AsyncMemoryHint
    Value: 0

    Key  : Hypervisor.Flags.CoreSchedulerRequested
    Value: 0

    Key  : Hypervisor.Flags.CpuManager
    Value: 1

    Key  : Hypervisor.Flags.DeprecateAutoEoi
    Value: 0

    Key  : Hypervisor.Flags.DynamicCpuDisabled
    Value: 1

    Key  : Hypervisor.Flags.Epf
    Value: 0

    Key  : Hypervisor.Flags.ExtendedProcessorMasks
    Value: 1

    Key  : Hypervisor.Flags.HardwareMbecAvailable
    Value: 1

    Key  : Hypervisor.Flags.MaxBankNumber
    Value: 0

    Key  : Hypervisor.Flags.MemoryZeroingControl
    Value: 0

    Key  : Hypervisor.Flags.NoExtendedRangeFlush
    Value: 0

    Key  : Hypervisor.Flags.NoNonArchCoreSharing
    Value: 1

    Key  : Hypervisor.Flags.Phase0InitDone
    Value: 1

    Key  : Hypervisor.Flags.PowerSchedulerQos
    Value: 0

    Key  : Hypervisor.Flags.RootScheduler
    Value: 0

    Key  : Hypervisor.Flags.SynicAvailable
    Value: 1

    Key  : Hypervisor.Flags.UseQpcBias
    Value: 0

    Key  : Hypervisor.Flags.Value
    Value: 4853999

    Key  : Hypervisor.Flags.ValueHex
    Value: 4a10ef

    Key  : Hypervisor.Flags.VpAssistPage
    Value: 1

    Key  : Hypervisor.Flags.VsmAvailable
    Value: 1

    Key  : Hypervisor.RootFlags.AccessStats
    Value: 1

    Key  : Hypervisor.RootFlags.CrashdumpEnlightened
    Value: 1

    Key  : Hypervisor.RootFlags.CreateVirtualProcessor
    Value: 1

    Key  : Hypervisor.RootFlags.DisableHyperthreading
    Value: 0

    Key  : Hypervisor.RootFlags.HostTimelineSync
    Value: 1

    Key  : Hypervisor.RootFlags.HypervisorDebuggingEnabled
    Value: 0

    Key  : Hypervisor.RootFlags.IsHyperV
    Value: 1

    Key  : Hypervisor.RootFlags.LivedumpEnlightened
    Value: 1

    Key  : Hypervisor.RootFlags.MapDeviceInterrupt
    Value: 1

    Key  : Hypervisor.RootFlags.MceEnlightened
    Value: 1

    Key  : Hypervisor.RootFlags.Nested
    Value: 0

    Key  : Hypervisor.RootFlags.StartLogicalProcessor
    Value: 1

    Key  : Hypervisor.RootFlags.Value
    Value: 1015

    Key  : Hypervisor.RootFlags.ValueHex
    Value: 3f7

    Key  : SecureKernel.HalpHvciEnabled
    Value: 0

    Key  : WER.OS.Branch
    Value: vb_release

    Key  : WER.OS.Version
    Value: 10.0.19041.1


BUGCHECK_CODE:  d1

BUGCHECK_P1: 8

BUGCHECK_P2: 2

BUGCHECK_P3: 0

BUGCHECK_P4: fffff8068a1c75b0

FILE_IN_CAB:  MEMORY_2.DMP

READ_ADDRESS: unable to get nt!PspSessionIdBitmap
 0000000000000008 

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

DPC_STACK_BASE:  FFFFF80634CB6FB0

TRAP_FRAME:  fffff80634cb6700 -- (.trap 0xfffff80634cb6700)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=0000000000000002 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=fffff8068a1c75b0 rsp=fffff80634cb6890 rbp=ffffd80c250e08b0
 r8=ffffd80c28de9530  r9=ffffd80c28de9530 r10=fffff806318d7910
r11=0000000000000002 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei ng nz na pe nc
tunnel!TeredoWfpRemoveHashEntry+0x2c:
fffff806`8a1c75b0 48395a08        cmp     qword ptr [rdx+8],rbx ds:00000000`00000008=????????????????
Resetting default scope

STACK_TEXT:  
fffff806`34cb65b8 fffff806`31a09169     : 00000000`0000000a 00000000`00000008 00000000`00000002 00000000`00000000 : nt!KeBugCheckEx
fffff806`34cb65c0 fffff806`31a05469     : fffff806`34cb6780 01800040`06aff7c8 ffffd80c`00000000 fffff806`899934cf : nt!KiBugCheckDispatch+0x69
fffff806`34cb6700 fffff806`8a1c75b0     : fffff806`8a1ca590 ffffd80c`28de9530 ffffd80c`07109680 fffff806`89992d5e : nt!KiPageFault+0x469
fffff806`34cb6890 fffff806`8a1ca615     : ffffd80c`28de9530 ffffd80c`07d31d50 00000000`c000000d 00000000`00000000 : tunnel!TeredoWfpRemoveHashEntry+0x2c
fffff806`34cb68c0 fffff806`8a1ca5e3     : fffff806`8a1ca590 ffffd80b`f90c3010 00000000`00000132 ffffd80c`07109680 : tunnel!TeredoWfpDereferenceFlowContextImp+0x1d
fffff806`34cb68f0 fffff806`33fa0577     : ffffd80b`f90c3010 fffff806`8a00ff02 ffffd80b`f912dfa0 fffff806`33e15a4b : tunnel!TeredoWfpDdLayerDelete+0x53
fffff806`34cb6920 fffff806`33fa0037     : 00000000`00027f9f ffffd80c`09b6c040 ffffd80c`09b6c218 ffffd80b`f912dfa0 : NETIO!WfpNotifyFlowContextDelete+0x20b
fffff806`34cb69a0 fffff806`340eb580     : 00000000`0000ff02 ffffd80c`250e08b0 fffff806`34cb6b80 ffffd80b`f9877c50 : NETIO!KfdAleNotifyFlowDeletion+0x1c7
fffff806`34cb6a00 fffff806`340b929a     : ffffe4ea`f64b8f50 00000000`00000000 ffffd80c`09b6c268 fffff806`34cb6b80 : tcpip!WfpAleFreeRemoteEndpoint+0x30
fffff806`34cb6a80 fffff806`340b7407     : ffffd80c`09b6c078 ffffd80c`09b6c268 fffff806`34cb6b80 fffff806`33e1432e : tcpip!WfpAleDecrementWaitRef+0x76
fffff806`34cb6ab0 fffff806`340b7358     : ffffd80c`09b6c218 00000000`00000180 fffff806`34cb6b70 00000000`00000002 : tcpip!WfpAleDeleteRemoteEndpointLru+0x9b
fffff806`34cb6b00 fffff806`3189a24e     : fffff806`2b571240 fffff806`34cb6e70 fffff806`2b56e180 fffff806`00000002 : tcpip!LruCleanupDpcRoutine+0x378
fffff806`34cb6bb0 fffff806`31899534     : fffff806`2b56e180 00000000`00000000 00000000`0000002a 00000000`003bc4ce : nt!KiExecuteAllDpcs+0x30e
fffff806`34cb6d20 fffff806`319fe1f5     : 00000000`00000000 fffff806`2b56e180 00000000`00000000 fffff806`34263000 : nt!KiRetireDpcList+0x1f4
fffff806`34cb6fb0 fffff806`319fdfe0     : fffff806`34263000 ffffd80c`0bf6f4b0 00000000`00000000 fffff806`340e9343 : nt!KxRetireDpcList+0x5
fffffe83`6ceba200 fffff806`319fd6ae     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : nt!KiDispatchInterruptContinue
fffffe83`6ceba230 fffff806`3183a23b     : ffffd80b`f99058e0 fffffe83`6ceba640 00000000`00000000 00000000`00000017 : nt!KiDpcInterrupt+0x2ee
fffffe83`6ceba3c0 fffff806`340a4e94     : ffffd80b`f99059c8 00000000`00000000 00000000`6ceba600 ffffd80b`f99058e8 : nt!KzLowerIrql+0x1b
fffffe83`6ceba3f0 fffff806`34100026     : fffff806`34263000 fffffe83`6ceba640 fffff806`34263000 ffffd80b`f9979bf0 : tcpip!IppSendDatagramsCommon+0x4c4
fffffe83`6ceba570 fffff806`340ffc16     : ffffd80c`052e3930 ffffb652`d08e4538 fffff806`32325440 00000000`0003226e : tcpip!IppProcessMulticastDiscoveryTimeoutEvents+0x3ca
fffffe83`6ceba9b0 fffff806`31874825     : ffffd80b`f909c8e0 ffffd80b`f909c8e0 ffffd80b`f987ca60 fffff806`45675440 : tcpip!IppMulticastWorkerRoutine+0xa6
fffffe83`6cebaa00 fffff806`318b8515     : ffffd80c`204a6040 ffffd80c`204a6040 fffff806`318746f0 00000000`00000000 : nt!IopProcessWorkItem+0x135
fffffe83`6cebaa70 fffff806`31955855     : ffffd80c`204a6040 00000000`00000080 ffffd80b`f5902080 00000000`00000001 : nt!ExpWorkerThread+0x105
fffffe83`6cebab10 fffff806`319fe808     : ffff8981`b8200180 ffffd80c`204a6040 fffff806`31955800 47c05923`a111f1f4 : nt!PspSystemThreadStartup+0x55
fffffe83`6cebab60 00000000`00000000     : fffffe83`6cebb000 fffffe83`6ceb4000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x28


SYMBOL_NAME:  tunnel!TeredoWfpRemoveHashEntry+2c

MODULE_NAME: tunnel

IMAGE_NAME:  tunnel.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  2c

FAILURE_BUCKET_ID:  AV_tunnel!TeredoWfpRemoveHashEntry

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {ed81a04d-9dd2-aa51-bc2a-d6f45f90b7a5}

Followup:     MachineOwner
---------
```

The root cause of the crash is likely related to an invalid memory access in the **`tunnel.sys`** driver. The function **`TeredoWfpRemoveHashEntry`** attempted to read from a null pointer at an elevated IRQL, which caused a bugcheck. This is likely a **code defect** in a **specific version** of the **`tunnel.sys`** driver. We could try to update this driver and see if the issue will then get resolved.

```
0: kd> !mex.mods tunnel
Number of modules: loaded: 263 unloaded: 50 
Base             End                Flags   Time stamp            CLR   Module name   Version   Path   Filters:[ Interest|3rd|K|U|A ] More...
=======================================================================================================================================
fffff8068a1c0000 fffff8068a1ea000 |       | 2018-09-22 13:54:37 |  No | tunnel      | 0.0.0.0 | \SystemRoot\System32\drivers\tunnel.sys
```

