# Description

This write-up will present a case study of using ETW (Event Tracing for Windows) to analyze an active Cobalt Strike Beacon that was still active and communicating to it's C2 Server. The study began when I was analyzing several memory dumps that were associated with a Cobalt Strike Beacon. 

Link to all the ETL files can be downloaded here: https://mega.nz/file/u9V31KiY#xWVRVeqe8qJ8rB0ccKHFjAh4iIhpfFHlyFbldRgCX7s

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a31ddfe3-ce1c-4e79-a2a8-8a0b591ea035)


The first step in my analysis involved examining the unique call stacks of each memory dump. I noticed that most of them shared a very similar call stack, as illustrated below:

```
1 thread [stats]: 0
    00007ff893facdf4 ntdll!NtWaitForSingleObject+0x14
    00007ff891971a8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff87eeaf4b8 wininet!CPendingSyncCall::HandlePendingSync_AppHangIsAppBugForCallingWinInetSyncOnUIThread+0xe0
    00007ff87eeaa10f wininet!INTERNET_HANDLE_OBJECT::HandlePendingSync+0x33
    00007ff87ee50a7e wininet!HttpWrapSendRequest+0xa476e
    00007ff87ee20ff5 wininet!InternalHttpSendRequestA+0x5d
    00007ff87ee20f68 wininet!HttpSendRequestA+0x58
    0000024fa0d01271 0x24fa0d01271
    0000024fa0a4d7d0 0x24fa0a4d7d0
    000000b4dd9afa20 0xb4dd9afa20
    0000000056a2b5f0 0x56a2b5f0
    0000024fa0c0dbd0 0x24fa0c0dbd0

1 thread [stats]: 5
    0000000000000000 0x0

4 threads [stats]: 1 2 3 4
    00007ff893fb07c4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff893f62dc7 ntdll!TppWorkerThread+0x2f7
    00007ff893be7034 kernel32!BaseThreadInitThunk+0x14
    00007ff893f62651 ntdll!RtlUserThreadStart+0x21
```

While there were slight variations among them, I still observed the presence of **`wininet`** functions within the call stacks. This library is part of the Windows Internet (WinINet) API, which provides applications the ability to access standard Internet protocols such as HTTP. The presence of these **`wininet`** functions indicates that the Beacon is performing some network operations.


```
1 thread [stats]: 0
    00007ff893fad3f4 ntdll!NtDelayExecution+0x14
    00007ff89199967e KERNELBASE!SleepEx+0x9e
    000000000040305f artifact_x64+0x305f
    00000000004013b4 artifact_x64+0x13b4
    00000000004014db artifact_x64+0x14db
    00007ff893be7034 kernel32!BaseThreadInitThunk+0x14
    00007ff893f62651 ntdll!RtlUserThreadStart+0x21

1 thread [stats]: 2
    00007ff893facdf4 ntdll!NtWaitForSingleObject+0x14
    00007ff891971a8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff87eeaf4b8 wininet!CPendingSyncCall::HandlePendingSync_AppHangIsAppBugForCallingWinInetSyncOnUIThread+0xe0
    00007ff87eeaa10f wininet!INTERNET_HANDLE_OBJECT::HandlePendingSync+0x33
    00007ff87ee50a7e wininet!HttpWrapSendRequest+0xa476e
    00007ff87ee20ff5 wininet!InternalHttpSendRequestA+0x5d
    00007ff87ee20f68 wininet!HttpSendRequestA+0x58
    0000000000020169 0x20169
    00000000000201d6 0x201d6
    000000000002000a 0x2000a
    0000000000cc000c 0xcc000c
    0000000000020169 0x20169
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    000000000002000a 0x2000a
    0000000000cc000c 0xcc000c
    0000000000020169 0x20169
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    000000000002000a 0x2000a
    0000000000cc000c 0xcc000c
    0000000000020169 0x20169
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    000000000002000a 0x2000a
    0000000000cc000c 0xcc000c
    0000000000020169 0x20169
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    000000000002000a 0x2000a
    0000000000cc000c 0xcc000c
    0000000000020169 0x20169
    00000000000201d6 0x201d6
    00000000000201d6 0x201d6
    0000000000020186 0x20186
    000000000002000a 0x2000a

1 thread [stats]: 9
    0000000000000000 0x0

7 threads [stats]: 1 3 4 5 6 7 8
    00007ff893fb07c4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff893f62dc7 ntdll!TppWorkerThread+0x2f7
    00007ff893be7034 kernel32!BaseThreadInitThunk+0x14
    00007ff893f62651 ntdll!RtlUserThreadStart+0x21
```

As discussed previously, we were able to observe the presence of **`wininet`** functions within the call stacks. For example, **`HttpSendRequestA`** is a public WinINet API function used to send an HTTP request. This is also documented on MSDN and can be found here: https://learn.microsoft.com/en-us/windows/win32/api/wininet/nf-wininet-httpsendrequesta

**`HttpSendRequestA`** is a public WinINet API function that is originating from **`wininet.dll`**. **`wininet.dll`** is a system file that is part of the Microsoft Windows Internet (WinINet) API. Given that ETW Providers typically begin with the prefix **`Microsoft-`**. I've decided to run strings against **`wininet.dll`** to look and determine if it includes any strings that look like ETW Providers. 

For this example, I utilized **`bstrings.exe`** by Eric Zimmerman, although you're free to use any tool of your preference. It summarizes that 7 strings were found and it was able to identify the ETW providers within the **`wininet.dll`** library

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b23decc8-f36d-4008-832f-0b2d096fec5b)


Further one, with this information. I've decided to look at the Manifest of this ETW Provider to get a better understanding of the structure. A manifest in the context of ETW is an XML file that outlines the events, their IDs, messages, and related metadata, that an ETW provider can produce for diagnostic purposes. Here is a short snippet:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c0cc390a-3449-49ae-91a6-8edc78835d92)


To acquire a basic understanding and a bit more detailed description, we can use the **`logman.exe`** utility to print basic information about a particular ETW Provider:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/81d610b6-b342-4023-86f1-ed57ed6612eb)

# How to view which ETW Providers a Process is sending events to?

One of the important thing is viewing all the ETW providers that a specific process is sending events to, so it helps to narrow down on the one's we would like to trace. To achieve this, we can use the **`logman.exe`** utility and filter on the PID of our process of interest:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4c076e0d-7367-41c2-82f1-c838988047b5)

# How to capture an ETW trace?

This batch file is to start and stop an Event Tracing for Windows (ETW) trace using the netsh command-line tool. This file serves as a template for tracing ETW Providers of interest. In this particular example, we are tracing the following ETW Providers:

- **Microsoft-Windows-WinHttp**
- **Microsoft-Windows-WinINet**

We are setting the maximum size of the trace file to 4096 MB and allow the trace file to overwrite existing files. Save the following as an **batch** file:

```
@echo off
:: Start the ETW Trace using netsh
netsh trace start capture=yes overwrite=yes maxsize=4096 tracefile=C:\ETL\ETWTrace.etl ^
provider="Microsoft-Windows-WinHttp" keywords=0xffffffffffffffff level=0xff ^
provider="Microsoft-Windows-WinINet" keywords=0xffffffffffffffff level=0xff

@echo ETW Trace started successfully.
ECHO Reproduce your issue and enter any key to stop tracing.

@echo on
pause

:: Stop the ETW Trace
netsh trace stop
@echo ETW Trace stopped successfully.
```

Run the **batch** file with **Administrator** privileges, then launch the Cobalt Strike Beacon and wait until it establishes a connection to its Command and Control (C2) server.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ac14979-5603-4191-abd7-6a96e476c135)


Here we can see that our Cobalt Strike Beacon is running fine and establishing the C2 connection:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a36b55b7-cb9a-4e2d-8113-f04f745e20d8)


After the Beacon has successfully run and established a connection to its Command and Control (C2), we can stop the tracing. This will then create an ETW trace for us which is an file with the **`.etl`** extension.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/55050cf8-cd01-406b-a326-092c398850dd)

# Loading ETL File in WPA

Load the **.etl** file in **Windows Performance Analyzer**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b69081c2-47e4-4678-8339-c464fe1de763)


Expand the **System Activity** and click on **Generic Events**. From there on, select **Add graph to Analysis View**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5a1b4bbd-7474-4ec1-aca0-9606117e2854)


In the **View Editor**, choose to display the columns for **Process Id** and **Message**, as these are useful for our purposes. Other columns, such as **CPU** or **Thread Id**, can be removed for the time being as they are not immediately relevant.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ea217fb4-5628-4591-a72b-d0fa34e631e7)


This is a similar view that we now should see in **Windows Performance Analyzer:**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b06c75aa-e714-49c6-a051-bad64eef97f2)


# WPA Walk through - Analyzing ETL

When analyzing the **`.etl`** file, one of the first things I noticed within the **`Microsoft-Windows-WinINet`** ETW Provider was the task **`Task.WININET_HTTPS_SERVER_CERT_VALIDATED`** with Event ID **705**. This task represents events related to the validation of an HTTPS server's certificate.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b9381dfa-31c7-4803-a135-5ad683447177)


The schema of this Event ID **705** looks like the following:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ae0f68a7-449e-4161-a2bb-fc90930d208b)


The **`CertHash`** field is in particularly valuable because it contains a unique identifier for the certificate and to verify its integrity. Cobalt Strike Team Servers are using a digital certificate to establish a secure SSL/TLS connection between the Cobalt Strike server. The default certificate provided with Cobalt Strike is identifiable and often not changed by Threat Actors, which makes it easy to identify that the malware of interest is Cobalt Strike.

**`6ECE5ECE4192683D2D84E25B0BA7E04F9CB7EB7C`** is a SHA-1 hash that is associated with the default certificate used by Cobalt Strike Team Servers. This hash is part of the digital certificate's details that uniquely identifies it. Since this default certificate and its hash are well-known, it can be used as an IOC. In other words, every time when we see this hash in our ETW trace. It's very likely that the malware is a Cobalt Strike Beacon.

To change the default certificate of the Cobalt Strike Team Server is pretty straightforward and not hard. Below is a sample of a self-signed certificate in use. Possessing the SHA1 details of a Cobalt Strike Team Server can be beneficial for Threat Intelligence purposes. This data allows for the hunting of additional potential team servers using tools like Shodan or Censys.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1e72d392-7a69-4339-96e8-a6e3b3903697)


This particular Event ID is produced when, for instance, a Cobalt Strike Beacon successfully establishes an HTTPS connection with its Command and Control (C2) server, and is able to validate it's SSL/TLS certificate. If the C2 server happens to be offline or using unencrypted traffic (e.g. HTTP) during the time that the Beacon is traced, this Event ID will not be generated.

Another valuable artifact is the **`Wininet_UsageLogRequest`** task within the **`Microsoft-Windows-WinINet`** ETW Provider. This task is associated with the Event ID **1057** and is related to HTTP requests that are being made.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b38a73fe-04af-4463-a18d-d014a02f3fb7)


The schema of this Event ID **1057** looks like the following:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/04b1feb4-7952-4e24-96e9-6c72109ebc59)


The **`RequestHeaders`** field shows HTTP request headers, which in this case contain **GET** request for a specific **URI**. The observed GET request (**`/fwlink`**) is part of a communication pattern that is related to Cobalt Strike's Malleable C2 profiles. There are a set of URIs that the Cobalt Strike Beacon will use for HTTP GET requests as part of its command and control (C2) communications. The Beacon will by default randomly choose from this specified pool of URIs when making these requests, but keep in mind that this can be easily changed. Palo Alto Unit42 has covered that here: https://unit42.paloaltonetworks.com/cobalt-strike-malleable-c2-profile/

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0ab1d835-703b-495d-b5cd-45ec8598077a)



We can inspect the **`RequestHeaders`** field more closer to get some additional fields and this is how it could look like. The **`Cookie`** header in the GET request contains a long string that looks like a Base64-encoded value. This string is encrypted metadata sent by the Cobalt Strike Beacon to the team server.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8b7f04c9-2b78-4f4f-abae-a3d7b315b51f)


To summarize everything; the **`Microsoft-Windows-WinINet`** ETW Provider proves to be valuable provider for an initial triage. The **SHA1** hash referenced in this write-up is the standard certificate used by a Cobalt Strike Team Server, making it a reliable indicator. Additionally, the presence of GET/POST requests to highly **specific URIs** that align with the default settings of Cobalt Strike's Malleable C2 profiles, along with the presence of a **`Cookie`** header in the HTTP requests that looks similar to a Base64 string, is another strong indication to consider as well.

# Use-cases of Event ID 705 not being generated

We did mentioned that Event ID **705** is generated when a Cobalt Strike Beacon can establish a successful (encrypted traffic) HTTPS connection with it's C2 server to validate it's certificate. However, how does it look like when it could NOT establish the connection?

In this example, Event ID **705** is not generated because the Cobalt Strike C2 server is offline. As a result, Event ID **302** is triggered, which indicates a failure to establish a connection with the C2 server.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/96c9a02c-f2af-40a1-9306-030bbefe35a7)


The log entries indicate repeated attempts to establish a TCP connection with the server at IP address **121[.]4.67.78**, but the connection couldn't be established since the C2 server is offline:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/34c1bd4b-377f-4bfc-99c3-0137245e8865)


The second example is when an Threat Actor is using an unencrypted traffic for it's Cobalt Strike Beacon. From Event ID **1075** we are able to observe that an HTTP request is being made rather than HTTPS:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3d068d55-073e-4552-822b-0124e2e5bc86)


This particular Event ID does have an field which is **`URL`** that shows the request being made to the exact URL. Each request successfully received a response with the status **`HTTP/1.1 200 OK`**, indicating that the requests were successfully processed by the server. The request contains a **specific URI** that is listed as a default Cobalt Strike's Malleable C2 profile. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b17f1fdf-c575-4c69-add1-df7dc63223f9)


# References

- **Default Cobalt Strike Team Server Certificate:** https://www.elastic.co/guide/en/security/current/default-cobalt-strike-team-server-certificate.html
- **Cobalt Strike Analysis and Tutorial: How Malleable C2 Profiles Make Cobalt Strike Difficult to Detect:** https://unit42.paloaltonetworks.com/cobalt-strike-malleable-c2-profile/
- **Cobalt Strike: Using Known Private Keys To Decrypt Traffic – Part 2** https://blog.nviso.eu/2021/10/27/cobalt-strike-using-known-private-keys-to-decrypt-traffic-part-2/
- **Understanding Cobalt Strike Profiles - Updated for Cobalt Strike 4.6:** https://blog.zsec.uk/cobalt-strike-profiles/
- **BStrings:** https://www.sans.org/tools/bstrings/
- **System Informer:** https://systeminformer.sourceforge.io/downloads.php
