# Summary

Customer reported that when attempting to deploy a printer using Group Policy in the Print Management MMC console, the associated Group Policy Object (GPO) does not appear in the 'Deployed Printers' section, preventing successful deployment. The issue occurred unexpectedly, as the customer was previously able to deploy printers without any problems. It was also noted that both clients and servers were claimed to be fully patched at the time.

![image](https://github.com/user-attachments/assets/62591211-e506-4bf7-98c0-54655bf31d8d)




## Analysis

The network trace reveals LDAP traffic with a `sizeLimitExceeded` result code, indicating that the LDAP query is returning more results than the server's configured limit allows. The query for `PushedPrinterConnections` exceeds this limit, resulting in an incomplete response. The client is unable to process the partial result, which likely causes the blank screen symptom in the Print Management MMC.

![image](https://github.com/user-attachments/assets/1a7b3b21-be5f-47d7-8fea-d61ec0852c7b)


As previously discussed, the network trace indicates that the LDAP traffic is exceeding the limit. Let's use Time Travel Debugging to confirm this further. After loading the **MMC01.run** trace file in WinDbg, we begin by using the g command. This command **(Go forward)** allows the debugger to run until the end of the trace

```
0:000> g
ModLoad: 00007fff`d2720000 00007fff`d28d5000   C:\windows\SYSTEM32\DUI70.dll
ModLoad: 00007fff`e0860000 00007fff`e0ad9000   C:\windows\WinSxS\amd64_microsoft.windows.common-controls_6595b64144ccf1df_6.0.17763.3650_none_de72e6e05349b85a\Comctl32.dll
ModLoad: 00007fff`e8ed0000 00007fff`e9179000   C:\windows\SYSTEM32\iertutil.dll
ModLoad: 00007fff`eb560000 00007fff`eb586000   C:\windows\SYSTEM32\srvcli.dll
ModLoad: 00007fff`ef6a0000 00007fff`ef6ae000   C:\windows\SYSTEM32\netutils.dll
ModLoad: 00007fff`efa00000 00007fff`efa0c000   C:\windows\SYSTEM32\CRYPTBASE.DLL
ModLoad: 00007fff`e69c0000 00007fff`e6b97000   C:\windows\SYSTEM32\urlmon.dll
ModLoad: 00007fff`f2410000 00007fff`f24b9000   C:\windows\System32\clbcatq.dll
ModLoad: 00007fff`d2450000 00007fff`d2718000   C:\windows\system32\mmcndmgr.dll
ModLoad: 00007fff`f1c90000 00007fff`f1dfb000   C:\windows\System32\MSCTF.dll
ModLoad: 00007fff`f0040000 00007fff`f0052000   C:\windows\System32\MSASN1.dll
ModLoad: 00007fff`f0ec0000 00007fff`f10b5000   C:\windows\System32\CRYPT32.dll
ModLoad: 00007fff`ee6a0000 00007fff`ee6ce000   C:\windows\SYSTEM32\dwmapi.dll
ModLoad: 00007fff`d9df0000 00007fff`d9e5c000   C:\Windows\System32\oleacc.dll
ModLoad: 00007fff`e1800000 00007fff`e18a9000   C:\windows\WinSxS\amd64_microsoft.windows.common-controls_6595b64144ccf1df_5.82.17763.3650_none_6d0994ae59f6ca6b\COMCTL32.dll
ModLoad: 00007fff`e3970000 00007fff`e39b1000   C:\windows\system32\mlang.dll
ModLoad: 00007fff`eed50000 00007fff`eee12000   C:\windows\system32\dxgi.dll
ModLoad: 00007fff`ecac0000 00007fff`ecd3e000   C:\windows\system32\d3d11.dll
ModLoad: 00007fff`ed760000 00007fff`ed923000   C:\windows\system32\dcomp.dll
ModLoad: 00007fff`d9d90000 00007fff`d9de6000   C:\windows\system32\dataexchange.dll
ModLoad: 00007fff`ee900000 00007fff`ee928000   C:\windows\system32\RMCLIENT.dll
ModLoad: 00007fff`ee6d0000 00007fff`ee8dc000   C:\windows\system32\twinapi.appcore.dll
ModLoad: 00007fff`eb620000 00007fff`eb65a000   C:\windows\system32\xmllite.dll
ModLoad: 00007fff`d4cd0000 00007fff`d4cdd000   C:\windows\SYSTEM32\atlthunk.dll
ModLoad: 00007fff`eb4c0000 00007fff`eb4ca000   C:\windows\SYSTEM32\VERSION.dll
ModLoad: 00007fff`e1d50000 00007fff`e1d95000   C:\windows\SYSTEM32\edputil.dll
ModLoad: 00007fff`ef010000 00007fff`ef041000   C:\windows\SYSTEM32\ntmarta.dll
ModLoad: 00007fff`ee010000 00007fff`ee0f1000   C:\windows\System32\CoreMessaging.dll
ModLoad: 00007fff`ebf60000 00007fff`ec0b1000   C:\windows\SYSTEM32\wintypes.dll
ModLoad: 00007fff`ec0c0000 00007fff`ec3e2000   C:\windows\System32\CoreUIComponents.dll
ModLoad: 00007fff`ec5a0000 00007fff`ec633000   C:\windows\System32\TextInputFramework.dll
ModLoad: 00007fff`f1450000 00007fff`f1576000   C:\windows\System32\COMDLG32.dll
ModLoad: 00007fff`ec3f0000 00007fff`ec599000   C:\windows\SYSTEM32\PROPSYS.dll
ModLoad: 00007fff`f0130000 00007fff`f0156000   C:\windows\System32\bcrypt.dll
ModLoad: 00007fff`c5180000 00007fff`c5600000   C:\windows\SYSTEM32\msi.dll
ModLoad: 00007fff`e70b0000 00007fff`e70cc000   C:\windows\system32\ATL.DLL
ModLoad: 00007fff`ef580000 00007fff`ef5c1000   C:\windows\system32\logoncli.dll
ModLoad: 00007fff`f13f0000 00007fff`f1450000   C:\windows\System32\WLDAP32.dll
ModLoad: 00007fff`eb710000 00007fff`eb752000   C:\windows\system32\adsldpc.dll
ModLoad: 00007fff`eb760000 00007fff`eb7a5000   C:\windows\system32\ACTIVEDS.dll
ModLoad: 00007fff`f1360000 00007fff`f13cd000   C:\windows\System32\WS2_32.dll
ModLoad: 00007fff`e60d0000 00007fff`e60f8000   C:\windows\system32\NTDSAPI.dll
ModLoad: 00007fff`ebdc0000 00007fff`ebdca000   C:\windows\system32\DSROLE.dll
ModLoad: 00007fff`d2d50000 00007fff`d2dfd000   C:\windows\system32\dsuiext.dll
ModLoad: 00007fff`ef540000 00007fff`ef57d000   C:\windows\system32\IPHLPAPI.DLL
ModLoad: 00007fff`e1ba0000 00007fff`e1bce000   C:\windows\system32\dsprop.dll
ModLoad: 00007fff`e70e0000 00007fff`e70ed000   C:\windows\system32\DSPARSE.dll
ModLoad: 00007fff`eb660000 00007fff`eb678000   C:\windows\system32\samcli.dll
ModLoad: 00007fff`d2cb0000 00007fff`d2d47000   C:\windows\system32\CRYPTUI.dll
ModLoad: 00007fff`df910000 00007fff`df929000   C:\windows\system32\credui.dll
ModLoad: 00007fff`f1e00000 00007fff`f1e08000   C:\windows\System32\NSI.dll
ModLoad: 00007fff`ef5d0000 00007fff`ef696000   C:\windows\system32\DNSAPI.dll
ModLoad: 00007fff`e5ca0000 00007fff`e5cd0000   C:\windows\system32\WINBRAND.dll
ModLoad: 00007fff`efe30000 00007fff`efe3a000   C:\windows\system32\DPAPI.DLL
ModLoad: 00007fff`d2e00000 00007fff`d2efc000   C:\windows\system32\adprop.dll
ModLoad: 00007fff`df020000 00007fff`df059000   C:\windows\SYSTEM32\wincredui.dll
ModLoad: 00007fff`eb6b0000 00007fff`eb6bc000   C:\windows\system32\Secur32.dll
ModLoad: 00007fff`d2930000 00007fff`d2a42000   C:\windows\system32\dsadmin.dll
ModLoad: 00007fff`d1440000 00007fff`d1509000   C:\windows\system32\certca.dll
ModLoad: 00007fff`d1510000 00007fff`d182d000   C:\windows\system32\certenroll.dll
ModLoad: 00007fff`efb00000 00007fff`efb2c000   C:\windows\system32\ncrypt.dll
ModLoad: 00007fff`f03b0000 00007fff`f0411000   C:\windows\System32\WINTRUST.dll
ModLoad: 00007fff`f23f0000 00007fff`f240d000   C:\windows\System32\imagehlp.dll
ModLoad: 00007fff`d0ef0000 00007fff`d1435000   C:\windows\system32\ACLUI.dll
ModLoad: 00007fff`eefb0000 00007fff`eefd6000   C:\windows\system32\sppc.dll
ModLoad: 00007fff`eefe0000 00007fff`ef008000   C:\windows\system32\SLC.dll
ModLoad: 00007fff`d2140000 00007fff`d235e000   C:\windows\system32\certmgr.dll
ModLoad: 00007fff`efac0000 00007fff`efafc000   C:\windows\system32\NTASN1.dll
ModLoad: 00007fff`d4340000 00007fff`d4388000   C:\Windows\System32\mycomput.dll
ModLoad: 00007fff`f1810000 00007fff`f1c86000   C:\windows\System32\SETUPAPI.dll
ModLoad: 00007fff`e2860000 00007fff`e2869000   C:\windows\system32\WSOCK32.dll
ModLoad: 00007fff`eeca0000 00007fff`eecc2000   C:\windows\System32\GPAPI.dll
ModLoad: 00007fff`ef180000 00007fff`ef1cc000   C:\windows\System32\AUTHZ.dll
ModLoad: 00007fff`e1f80000 00007fff`e1f93000   C:\windows\System32\DSSEC.dll
ModLoad: 00007fff`ca650000 00007fff`ca6a0000   C:\windows\System32\framedynos.dll
ModLoad: 00007fff`d1b60000 00007fff`d1c78000   C:\windows\System32\GPEdit.dll
ModLoad: 00007fff`e1a60000 00007fff`e1a7c000   C:\Windows\System32\WINIPSEC.DLL
ModLoad: 00007fff`d1cc0000 00007fff`d1d44000   C:\Windows\System32\ipsmsnap.dll
ModLoad: 00007fff`d3b70000 00007fff`d3bc6000   C:\Windows\System32\POLSTORE.DLL
ModLoad: 00007fff`d1a80000 00007fff`d1b51000   C:\Windows\System32\ipsecsnp.dll
ModLoad: 00007fff`e68a0000 00007fff`e692f000   C:\windows\system32\WINSPOOL.DRV
ModLoad: 00007fff`eb860000 00007fff`eb879000   C:\windows\system32\NETAPI32.dll
ModLoad: 00007fff`dc430000 00007fff`dc466000   C:\windows\system32\puiapi.dll
ModLoad: 00007fff`d0bf0000 00007fff`d0cba000   C:\windows\system32\pmcsnap.dll
ModLoad: 00007fff`eb820000 00007fff`eb838000   C:\windows\System32\wkscli.dll
ModLoad: 00007fff`df7d0000 00007fff`df803000   C:\windows\System32\rasman.dll
ModLoad: 00007fff`d0a00000 00007fff`d0a82000   C:\windows\System32\MPRAPI.dll
ModLoad: 00007fff`eef10000 00007fff`eefa6000   C:\windows\System32\FirewallAPI.dll
ModLoad: 00007fff`d3ac0000 00007fff`d3b00000   C:\windows\System32\eappcfg.dll
ModLoad: 00007fff`de4c0000 00007fff`de4dc000   C:\windows\System32\RTRFILTR.dll
ModLoad: 00007fff`d0a90000 00007fff`d0be4000   C:\windows\System32\mprsnap.dll
ModLoad: 00007fff`eecd0000 00007fff`eecfd000   C:\windows\System32\fwbase.dll
ModLoad: 00007fff`ef0e0000 00007fff`ef134000   C:\windows\system32\SCECLI.dll
ModLoad: 00007fff`d0880000 00007fff`d09f7000   C:\windows\system32\wsecedit.dll
ModLoad: 00007fff`d07f0000 00007fff`d087f000   C:\windows\system32\filemgmt.dll
ModLoad: 00007fff`d0780000 00007fff`d07e5000   C:\Windows\System32\tapisnap.dll
ModLoad: 00007fff`e1c30000 00007fff`e1c3a000   C:\windows\system32\wbem\MMFUtil.DLL
ModLoad: 00007fff`d2af0000 00007fff`d2b48000   C:\windows\system32\wbem\wbemcntl.dll
ModLoad: 00007fff`d0700000 00007fff`d0776000   C:\Windows\System32\puiobj.dll
ModLoad: 00007fff`e0600000 00007fff`e0857000   C:\Windows\System32\msxml6.dll
ModLoad: 00007fff`dc500000 00007fff`dc62d000   C:\windows\system32\printui.dll
ModLoad: 00007fff`eb7d0000 00007fff`eb811000   C:\windows\system32\adsldp.dll
ModLoad: 00007fff`efe40000 00007fff`efedb000   C:\windows\SYSTEM32\sxs.dll
ModLoad: 00007fff`ef830000 00007fff`ef897000   C:\windows\system32\mswsock.dll
ModLoad: 00007fff`e7fc0000 00007fff`e7fca000   C:\windows\system32\wshqos.dll
ModLoad: 00007fff`e7fb0000 00007fff`e7fb8000   C:\windows\SYSTEM32\wshtcpip.DLL
ModLoad: 00007fff`e7e60000 00007fff`e7e68000   C:\windows\SYSTEM32\wship6.dll
ModLoad: 00007fff`e7670000 00007fff`e767a000   C:\Windows\System32\rasadhlp.dll
ModLoad: 00007fff`eb4e0000 00007fff`eb559000   C:\windows\System32\fwpuclnt.dll
ModLoad: 00007fff`ef8f0000 00007fff`ef9f5000   C:\windows\system32\kerberos.DLL
ModLoad: 00007fff`ef8a0000 00007fff`ef8b5000   C:\windows\SYSTEM32\cryptdll.dll
ModLoad: 00007fff`e9620000 00007fff`e971f000   C:\Windows\System32\WINHTTP.dll
ModLoad: 00007fff`c6fc0000 00007fff`c7c41000   C:\Windows\System32\ieframe.dll
ModLoad: 00007fff`e19f0000 00007fff`e1a42000   C:\windows\SYSTEM32\msIso.dll
ModLoad: 00007fff`cf080000 00007fff`d0700000   C:\Windows\System32\mshtml.dll
ModLoad: 00007fff`eb4d0000 00007fff`eb4da000   C:\windows\SYSTEM32\FLTLIB.DLL
ModLoad: 00007fff`eb5b0000 00007fff`eb5c3000   C:\windows\SYSTEM32\virtdisk.dll
ModLoad: 00007fff`e0110000 00007fff`e05f3000   C:\Windows\System32\WININET.dll
ModLoad: 00007fff`e1b70000 00007fff`e1b9a000   C:\Windows\System32\srpapi.dll
ModLoad: 00007fff`efa90000 00007fff`efab9000   C:\windows\SYSTEM32\WLDP.DLL
ModLoad: 00007fff`d1fb0000 00007fff`d213e000   C:\Windows\System32\ieapfltr.dll
ModLoad: 00007fff`e1f60000 00007fff`e1f72000   C:\windows\system32\msimtf.dll
ModLoad: 00007fff`cebd0000 00007fff`cf07f000   C:\Windows\System32\jscript9.dll
ModLoad: 00007fff`df730000 00007fff`df769000   C:\Windows\System32\msls31.dll
ModLoad: 00007fff`ed1a0000 00007fff`ed75e000   C:\Windows\System32\d2d1.dll
ModLoad: 00007fff`e4120000 00007fff`e4427000   C:\Windows\System32\DWrite.dll
ModLoad: 00007fff`e8060000 00007fff`e879b000   C:\windows\SYSTEM32\d3d10warp.dll
ModLoad: 00007fff`ee220000 00007fff`ee24d000   C:\windows\SYSTEM32\winmmbase.dll
ModLoad: 00007fff`ee250000 00007fff`ee274000   C:\windows\SYSTEM32\WINMM.dll
ModLoad: 00007fff`f2310000 00007fff`f2385000   C:\windows\System32\coml2.dll
TTD: End of trace reached.
(1ac.f5c): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 338942:0
ntdll!RtlLeaveCriticalSection+0x2a:
00007fff`f409404a f00fb14b08      lock cmpxchg dword ptr [rbx+8],ecx ds:00007fff`dea72420=fffffffe
```

Based on the network trace captured with Wireshark, we speculated that the issue might be related to an LDAP query exceeding a limit. To investigate further, the next step was to search for Win32 related functions responsible for querying operations.

![image](https://github.com/user-attachments/assets/4f786f0d-91cc-47f1-9ccc-1412641b77c4)



```
0:000> dx @$cursession.TTD.Calls("wldap32!ldap*").Select(c => c.Function).Distinct()
@$cursession.TTD.Calls("wldap32!ldap*").Select(c => c.Function).Distinct()                
    [0x0]            : WLDAP32!LdapClientInitialize
    [0x1]            : WLDAP32!LdapUsePrivateHeap
    [0x2]            : WLDAP32!ldapMalloc
    [0x3]            : WLDAP32!ldap_initW
    [0x4]            : WLDAP32!ldap_sslinitW
    [0x5]            : WLDAP32!LdapAllocateConnection
    [0x6]            : WLDAP32!LdapInitializeWinsock
    [0x7]            : WLDAP32!ldapFree
    [0x8]            : WLDAP32!LdapInitSecurity
    [0x9]            : WLDAP32!ldapWStringsIdentical
    [0xa]            : WLDAP32!ldap_dup_stringW
    [0xb]            : WLDAP32!LdapSetConnectionOption
    [0xc]            : WLDAP32!ldap_get_optionW
    [0xd]            : WLDAP32!LdapGetConnectionOption
    [0xe]            : WLDAP32!ldap_set_optionW
    [0xf]            : WLDAP32!ldap_connect
    [0x10]           : WLDAP32!LdapConnect
    [0x11]           : WLDAP32!LdapIsAddressNumeric
    [0x12]           : WLDAP32!LdapParallelConnect
    [0x13]           : WLDAP32!LdapWinsockBind
    [0x14]           : WLDAP32!LdapWakeupSelect
    [0x15]           : WLDAP32!ldap_bind_sW
    [0x16]           : WLDAP32!LdapBind
    [0x17]           : WLDAP32!LdapDetermineServerVersion
    [0x18]           : WLDAP32!ldap_search_ext_sW
    [0x19]           : WLDAP32!LdapSearch
    [0x1a]           : WLDAP32!LdapCreateRequest
    [0x1b]           : WLDAP32!LdapSaveSearchParameters
    [0x1c]           : WLDAP32!LdapEncodeFilter
    [0x1d]           : WLDAP32!LdapEncodeSimpleFilter
    [0x1e]           : WLDAP32!LdapSend
    [0x1f]           : WLDAP32!LdapSendRaw
    [0x20]           : WLDAP32!ldap_result_with_error
    [0x21]           : WLDAP32!LdapWaitForResponseFromServer
    [0x22]           : WLDAP32!LdapGetResponseFromServer
    [0x23]           : WLDAP32!LdapGetMessageWaitStructure
    [0x24]           : WLDAP32!LdapAllBuffersToMessages
    [0x25]           : WLDAP32!LdapBuffersToMessages
    [0x26]           : WLDAP32!LdapGetReceiveStructure
    [0x27]           : WLDAP32!LdapInitialDecodeMessage
    [0x28]           : WLDAP32!LdapFreeMessageWaitStructure
    [0x29]           : WLDAP32!ldap_first_record
    [0x2a]           : WLDAP32!LdapFirstAttribute
    [0x2b]           : WLDAP32!LdapGoToFirstAttribute
    [0x2c]           : WLDAP32!LdapNextAttribute
    [0x2d]           : WLDAP32!LdapGetValues
    [0x2e]           : WLDAP32!ldap_count_values
    [0x2f]           : WLDAP32!ldap_value_free
    [0x30]           : WLDAP32!ldap_msgfree
    [0x31]           : WLDAP32!LdapParseResult
    [0x32]           : WLDAP32!ldap_result2error
    [0x33]           : WLDAP32!LdapGetServiceNameForBind
    [0x34]           : WLDAP32!LdapSspiBind
    [0x35]           : WLDAP32!LdapConvertSecurityError
    [0x36]           : WLDAP32!LdapExchangeOpaqueToken
    [0x37]           : WLDAP32!LdapUnicodeToUTF8
    [0x38]           : WLDAP32!LdapGetLastError
    [0x39]           : WLDAP32!ldap_search_stW
    [0x3a]           : WLDAP32!ldap_MoveMemory
    [0x3b]           : WLDAP32!ldap_memfree
    [0x3c]           : WLDAP32!ldap_count_entries
    [0x3d]           : WLDAP32!ldap_count_records
    [0x3e]           : WLDAP32!ldap_first_entry
    [0x3f]           : WLDAP32!ldap_first_attributeW
    [0x40]           : WLDAP32!ldap_get_valuesW
    [0x41]           : WLDAP32!ldap_count_valuesW
    [0x42]           : WLDAP32!ldap_next_attributeW
    [0x43]           : WLDAP32!ldap_unbind
    [0x44]           : WLDAP32!LdapCheckControls
    [0x45]           : WLDAP32!LdapDupControls
    [0x46]           : WLDAP32!ldap_controls_freeW
    [0x47]           : WLDAP32!ldap_get_values_lenW
    [0x48]           : WLDAP32!LdapFreeReceiveStructure
    [0x49]           : WLDAP32!ldap_value_free_len
    [0x4a]           : WLDAP32!ldap_next_entry
    [0x4b]           : WLDAP32!ldap_next_record
```

`ldap_search_stW` is a Windows function used to perform LDAP searches, and it’s part of the `WLDAP32.dll` library, which helps applications interact with Active Directory. This function lets you query an LDAP directory by specifying a starting point (the search base), how deep to look (the scope, such as just one level or the whole subtree), and what you’re looking for (the search filter).

```
0:005> dx -g @$cursession.TTD.Calls("wldap32!ldap_search_stW").Select(c => new { Function = c.Function, TimeStart = c.TimeStart, SystemTimeStart = c.SystemTimeStart })
=========================================================================================================
=           = (+) Function               = (+) TimeStart = (+) SystemTimeStart                          =
=========================================================================================================
= [0x0]     - WLDAP32!ldap_search_stW    - 179D21:1E     - Wednesday, November 23, 2022 16:34:14.151    =
= [0x1]     - WLDAP32!ldap_search_stW    - 17F187:B4     - Wednesday, November 23, 2022 16:34:14.260    =
= [0x2]     - WLDAP32!ldap_search_stW    - 1802F9:9E     - Wednesday, November 23, 2022 16:34:14.292    =
= [0x3]     - WLDAP32!ldap_search_stW    - 18101C:9E     - Wednesday, November 23, 2022 16:34:14.292    =
= [0x4]     - WLDAP32!ldap_search_stW    - 1812B5:98     - Wednesday, November 23, 2022 16:34:14.307    =
= [0x5]     - WLDAP32!ldap_search_stW    - 181471:96     - Wednesday, November 23, 2022 16:34:14.307    =
= [0x6]     - WLDAP32!ldap_search_stW    - 181FB4:86     - Wednesday, November 23, 2022 16:34:14.338    =
= [0x7]     - WLDAP32!ldap_search_stW    - 19EE4B:A8     - Wednesday, November 23, 2022 16:34:15.245    =
= [0x8]     - WLDAP32!ldap_search_stW    - 19FE80:A8     - Wednesday, November 23, 2022 16:34:15.260    =
= [0x9]     - WLDAP32!ldap_search_stW    - 1A01CD:A8     - Wednesday, November 23, 2022 16:34:15.260    =
= [0xa]     - WLDAP32!ldap_search_stW    - 1A0517:A8     - Wednesday, November 23, 2022 16:34:15.276    =
= [0xb]     - WLDAP32!ldap_search_stW    - 1A0818:A8     - Wednesday, November 23, 2022 16:34:15.292    =
= [0xc]     - WLDAP32!ldap_search_stW    - 1A0B37:A8     - Wednesday, November 23, 2022 16:34:15.292    =
= [0xd]     - WLDAP32!ldap_search_stW    - 1A0E64:A8     - Wednesday, November 23, 2022 16:34:15.292    =
= [0xe]     - WLDAP32!ldap_search_stW    - 1A1190:A8     - Wednesday, November 23, 2022 16:34:15.307    =
= [0xf]     - WLDAP32!ldap_search_stW    - 1A42B6:A8     - Wednesday, November 23, 2022 16:34:15.370    =
= [0x10]    - WLDAP32!ldap_search_stW    - 1A47BA:A8     - Wednesday, November 23, 2022 16:34:15.385    =
= [0x11]    - WLDAP32!ldap_search_stW    - 1A519C:A8     - Wednesday, November 23, 2022 16:34:15.401    =
= [0x12]    - WLDAP32!ldap_search_stW    - 1A54CA:A8     - Wednesday, November 23, 2022 16:34:15.401    =
= [0x13]    - WLDAP32!ldap_search_stW    - 1A57C6:A8     - Wednesday, November 23, 2022 16:34:15.401    =
= [0x14]    - WLDAP32!ldap_search_stW    - 1A5AE5:A8     - Wednesday, November 23, 2022 16:34:15.417    =
= [0x15]    - WLDAP32!ldap_search_stW    - 1A5DFC:A8     - Wednesday, November 23, 2022 16:34:15.417    =
= [0x16]    - WLDAP32!ldap_search_stW    - 1A612F:A8     - Wednesday, November 23, 2022 16:34:15.417    =
= [0x17]    - WLDAP32!ldap_search_stW    - 2596E3:199    - Wednesday, November 23, 2022 16:34:30.584    =
= [0x18]    - WLDAP32!ldap_search_stW    - 25CD45:B4     - Wednesday, November 23, 2022 16:34:30.693    =
= [0x19]    - WLDAP32!ldap_search_stW    - 25F071:A8     - Wednesday, November 23, 2022 16:34:30.724    =
= [0x1a]    - WLDAP32!ldap_search_stW    - 25F51E:A8     - Wednesday, November 23, 2022 16:34:30.740    =
= [0x1b]    - WLDAP32!ldap_search_stW    - 25FA3A:A8     - Wednesday, November 23, 2022 16:34:30.740    =
= [0x1c]    - WLDAP32!ldap_search_stW    - 25FEF5:A8     - Wednesday, November 23, 2022 16:34:30.756    =
= [0x1d]    - WLDAP32!ldap_search_stW    - 26023E:A8     - Wednesday, November 23, 2022 16:34:30.756    =
= [0x1e]    - WLDAP32!ldap_search_stW    - 2605A0:A8     - Wednesday, November 23, 2022 16:34:30.756    =
= [0x1f]    - WLDAP32!ldap_search_stW    - 2608DD:A8     - Wednesday, November 23, 2022 16:34:30.773    =
= [0x20]    - WLDAP32!ldap_search_stW    - 260C2C:A8     - Wednesday, November 23, 2022 16:34:30.787    =
= [0x21]    - WLDAP32!ldap_search_stW    - 26327F:A8     - Wednesday, November 23, 2022 16:34:30.849    =
= [0x22]    - WLDAP32!ldap_search_stW    - 2635C1:A8     - Wednesday, November 23, 2022 16:34:30.865    =
= [0x23]    - WLDAP32!ldap_search_stW    - 263925:A8     - Wednesday, November 23, 2022 16:34:30.865    =
= [0x24]    - WLDAP32!ldap_search_stW    - 263CB6:A8     - Wednesday, November 23, 2022 16:34:30.881    =
= [0x25]    - WLDAP32!ldap_search_stW    - 263FD7:A8     - Wednesday, November 23, 2022 16:34:30.881    =
= [0x26]    - WLDAP32!ldap_search_stW    - 264339:A8     - Wednesday, November 23, 2022 16:34:30.881    =
= [0x27]    - WLDAP32!ldap_search_stW    - 264675:A8     - Wednesday, November 23, 2022 16:34:30.896    =
= [0x28]    - WLDAP32!ldap_search_stW    - 2649BF:A8     - Wednesday, November 23, 2022 16:34:30.896    =
= [0x29]    - WLDAP32!ldap_search_stW    - 2B81F3:197    - Wednesday, November 23, 2022 16:34:40.605    =
= [0x2a]    - WLDAP32!ldap_search_stW    - 2BBBE9:B4     - Wednesday, November 23, 2022 16:34:40.699    =
= [0x2b]    - WLDAP32!ldap_search_stW    - 2BDA7A:A8     - Wednesday, November 23, 2022 16:34:40.761    =
= [0x2c]    - WLDAP32!ldap_search_stW    - 2BDEA5:A8     - Wednesday, November 23, 2022 16:34:40.761    =
= [0x2d]    - WLDAP32!ldap_search_stW    - 2BE23C:A8     - Wednesday, November 23, 2022 16:34:40.777    =
= [0x2e]    - WLDAP32!ldap_search_stW    - 2BE551:A8     - Wednesday, November 23, 2022 16:34:40.777    =
= [0x2f]    - WLDAP32!ldap_search_stW    - 2BE83A:A8     - Wednesday, November 23, 2022 16:34:40.792    =
= [0x30]    - WLDAP32!ldap_search_stW    - 2BEB55:A8     - Wednesday, November 23, 2022 16:34:40.792    =
= [0x31]    - WLDAP32!ldap_search_stW    - 2BEE67:A8     - Wednesday, November 23, 2022 16:34:40.792    =
= [0x32]    - WLDAP32!ldap_search_stW    - 2BF180:A8     - Wednesday, November 23, 2022 16:34:40.808    =
= [0x33]    - WLDAP32!ldap_search_stW    - 2C11F8:A8     - Wednesday, November 23, 2022 16:34:40.870    =
= [0x34]    - WLDAP32!ldap_search_stW    - 2C1548:A8     - Wednesday, November 23, 2022 16:34:40.886    =
= [0x35]    - WLDAP32!ldap_search_stW    - 2C1863:A8     - Wednesday, November 23, 2022 16:34:40.886    =
= [0x36]    - WLDAP32!ldap_search_stW    - 2C1B73:A8     - Wednesday, November 23, 2022 16:34:40.886    =
= [0x37]    - WLDAP32!ldap_search_stW    - 2C1E5C:A8     - Wednesday, November 23, 2022 16:34:40.902    =
= [0x38]    - WLDAP32!ldap_search_stW    - 2C217B:A8     - Wednesday, November 23, 2022 16:34:40.902    =
= [0x39]    - WLDAP32!ldap_search_stW    - 2C248C:A8     - Wednesday, November 23, 2022 16:34:40.917    =
= [0x3a]    - WLDAP32!ldap_search_stW    - 2C279E:A8     - Wednesday, November 23, 2022 16:34:40.917    =
=========================================================================================================
```

Let’s navigate to the first call made to this function and view the call stack.

```
0:005> !tt 179D21:1E
Setting position: 179D21:1E
ModLoad: 00007fff`eb7d0000 00007fff`eb811000   C:\windows\system32\adsldp.dll
ModLoad: 00007fff`efe40000 00007fff`efedb000   C:\windows\SYSTEM32\sxs.dll
ModLoad: 00007fff`ef830000 00007fff`ef897000   C:\windows\system32\mswsock.dll
ModLoad: 00007fff`e7fc0000 00007fff`e7fca000   C:\windows\system32\wshqos.dll
ModLoad: 00007fff`e7fb0000 00007fff`e7fb8000   C:\windows\SYSTEM32\wshtcpip.DLL
ModLoad: 00007fff`e7e60000 00007fff`e7e68000   C:\windows\SYSTEM32\wship6.dll
ModLoad: 00007fff`e7670000 00007fff`e767a000   C:\Windows\System32\rasadhlp.dll
ModLoad: 00007fff`eb4e0000 00007fff`eb559000   C:\windows\System32\fwpuclnt.dll
ModLoad: 00007fff`ef8f0000 00007fff`ef9f5000   C:\windows\system32\kerberos.DLL
(1ac.1710): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 179D21:1E
WLDAP32!ldap_search_stW:
00007fff`f1401570 488bc4          mov     rax,rsp
```

This call stack gives us insight into the sequence of function calls leading up to the `ldap_search_stW` function.

```
0:005> k
 # Child-SP          RetAddr               Call Site
00 00000000`02bff258 00007fff`eb7174e6     WLDAP32!ldap_search_stW
01 00000000`02bff260 00007fff`eb717407     adsldpc!LdapSearchHelper+0x92
02 00000000`02bff2d0 00007fff`eb7e56c1     adsldpc!LdapSearchS+0x37
03 00000000`02bff320 00007fff`eb7e43e7     adsldp!CLDAPRootDSE::GetInfo+0x71
04 00000000`02bff370 00007fff`eb7e546d     adsldp!CPropertyCache::getproperty+0x7f
05 00000000`02bff3d0 00007fff`dc438bb2     adsldp!CLDAPRootDSE::Get+0x6d
06 00000000`02bff440 00007fff`dc4362e1     puiapi!TGlobalServiceDS::GetBSTRPropertyFromIADs+0x66
07 00000000`02bff490 00007fff`dc436b60     puiapi!TGlobalServiceDS::GetCurrentDomainDN+0x71
08 00000000`02bff4d0 00007fff`d0c22fd8     puiapi!TGlobalServiceDS::GetPPCsOnCurrentDomain+0xd0
09 00000000`02bff5f0 00007fff`d0c22ecf     pmcsnap!TPPCWorkItem::CollectPPCs+0x6c
0a 00000000`02bff630 00007fff`d0c53248     pmcsnap!TPPCWorkItem::Run+0x8f
0b 00000000`02bff660 00007fff`f40ae869     pmcsnap!NThreadingLibrary::TWorkCrew::tpSimpleCallback+0x28
0c 00000000`02bff690 00007fff`f4096964     ntdll!TppSimplepExecuteCallback+0x99
0d 00000000`02bff6e0 00007fff`f1f67974     ntdll!TppWorkerThread+0x644
0e 00000000`02bff9d0 00007fff`f40da371     KERNEL32!BaseThreadInitThunk+0x14
0f 00000000`02bffa00 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```

`puiapi!TGlobalServiceDS::GetPPCsOnCurrentDomain` is a function that is trying to fetch "Pushed Printer Connections" (PPCs) for the domain. This aligns with the observed LDAP query in the network trace that encountered the `sizeLimitExceeded` error. You can verify this yourself by loading `puiapi.dll` into IDA, navigating to the `TGlobalServiceDS::GetPPCsOnCurrentDomain` function, and reviewing the generated pseudocode.

![image](https://github.com/user-attachments/assets/43bda142-173d-4fd8-97f9-e9739ebe269c)


Let’s set a breakpoint on this function in the WinDbg Timeline view option. Once the breakpoint is hit, we can observe some interesting return values right away.

![image](https://github.com/user-attachments/assets/35dbd963-dcb7-4d21-9be6-9e72838ed144)


Once we’ve set the breakpoint, let’s display the event details to the first call made to this function.

```
0:005> dx @$cursession.TTD.Calls("puiapi!TGlobalServiceDS::GetPPCsOnCurrentDomain")[0x0]
@$cursession.TTD.Calls("puiapi!TGlobalServiceDS::GetPPCsOnCurrentDomain")[0x0]                
    EventType        : 0x0
    ThreadId         : 0x1710
    UniqueThreadId   : 0x7
    TimeStart        : 16F671:8F [Time Travel]
    TimeEnd          : 1A94EC:14D [Time Travel]
    Function         : puiapi!TGlobalServiceDS::GetPPCsOnCurrentDomain
    FunctionAddress  : 0x7fffdc436a90
    ReturnAddress    : 0x7fffd0c22fd8
    ReturnValue      : 0x80072023
    Parameters      
    SystemTimeStart  : Wednesday, November 23, 2022 16:34:13.854
    SystemTimeEnd    : Wednesday, November 23, 2022 16:34:15.495
```

Once we decode the value in the `ReturnValue` field, it aligns with the findings from the network trace. Specifically, this means the function's return value corresponds to the LDAP query's outcome, such as the `sizeLimitExceeded` error observed in the network capture.

```
0:005> !error 0x80072023
Error code: (HRESULT) 0x80072023 (2147950627) - The size limit for this request was exceeded.
```

## Workaround

This issue may occur due to the limitations set by Active Directory for LDAP queries, controlled by the `MaxPageSize` value, which defaults to 1000 entries. To address this, you can temporarily adjust this value. Instructions for making this temporary change can be found here: https://learn.microsoft.com/en-us/previous-versions/office/exchange-server-analyzer/aa998536(v=exchg.80)?redirectedfrom=MSDN

*"The MaxPageSize value of the LDAPAdminLimits attribute controls the number of records that can be returned for an LDAP query. The default number is 1,000 records"*

## Conclusion

The ADSI query error code `ERROR_DS_SIZELIMIT_EXCEEDED` is not displayed in the MMC console window, making it challenging to identify the root cause without additional debugging.
