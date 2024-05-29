# Description

A few months ago I was reading an interesting white paper of CrowdStrike talking about an IIS post-exploitation framework known as "IceApple" that was discovered by CrowdStrike Falcon OverWatch team. I found it interesting because it's a framework that operates purely in memory with a focus on having a minimal forensic footprint. The whitepaper can be found here: https://www.crowdstrike.com/wp-content/uploads/2022/05/crowdstrike-iceapple-a-novel-internet-information-services-post-exploitation-framework-1.pdf

CrowdStrike Falcon Overwatch detected the IceApple framework using a detection for reflective .NET assembly loads. This detection was first triggered by the loading of unusual .NET assemblies via byte arrays on a customer's Microsoft Exchange OWA server. Upon closer inspection, these assemblies were found to be suspicious due to their unusual loading method and naming inconsistencies.

The purpose of this write-up is to guide software engineers working with EDR solutions. It will focus on how to use specific ETW Providers to enhance visibility for .NET assemblies in their security solutions and how to build detections using the available telemetry. 

All the interest started when I was reading this in the whitepaper:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ccbe4c62-127b-4997-aedd-a7ddb2a0e9d7)

We can use this Batch file to trace this ETW Provider:

```
@echo off
:: Start the ETW Trace for Microsoft-Windows-DotNETRuntime
netsh trace start capture=yes overwrite=yes maxsize=4096 tracefile=C:\ETL\ETWTrace.etl ^
provider="Microsoft-Windows-DotNETRuntime" keywords=0xffffffffffffffff level=0xff

@echo ETW Trace started successfully.
ECHO Reproduce your issue and then press any key to stop tracing.

@echo on
pause

:: Stop the ETW Trace
netsh trace stop
@echo ETW Trace stopped successfully.
```

Link to all the **.etl** files can be found here: https://mega.nz/file/elFxBRBC#MowzT4az-57L4BgL1wOciQKtthqJcbjJDpboo9SJ4AU

# Telemetry

When we want to increase our visibility of .NET assembly loads, we could use ETW Providers such as **`Microsoft-Windows-DotNETRuntime`**. This provider is primarily used for collecting runtime information and diagnostics about .NET applications. It helps in analyzing performance, troubleshooting issues, but also in understanding the behavior of .NET processes through events like assembly loading, garbage collection, Just-In-Time (JIT) compilation, and other runtime events.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/86d9fa11-6e0f-4ac8-b9b9-de988142990f)

The **`CLRLoader`** task with Event ID **152** within the **`Microsoft-Windows-DotNETRuntime`** ETW provider is responsible for tracking the loading of Common Language Runtime (CLR) modules. This would typically include events related to the loading of .NET assemblies into an process.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/522aba06-7ae2-4611-a0ae-1ccc6f364d3c)

This output displays a log of .NET assemblies being loaded by the CLR, including their unique identifiers and paths, with some loaded from the Global Assembly Cache (GAC) and others from an Exchange Server directory.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/830cfe0d-3cf5-4fa5-907f-c87f9bcb96f9)

This event generates a lot of data, which is why some EDR vendors focus on collecting telemetry of .NET modules that are loaded reflectively, as these do not typically reference a file path, being instead loaded directly from a byte array in memory. This .NET assembly might for example be reflectively loaded because the **`ModuleILPath`** field lacks a traditional drive letter or network share path, which are typically present for assemblies loaded from a filesystem. The reason I use 'might' is that collecting Event ID **152** alone is not sufficient to conclusively determine this.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9e4c59ce-8962-4469-b7ac-8d748bbe61ef)


We would also need to collect Event ID **154** from **`Microsoft-Windows-DotNETRuntime`**. It shows a list of .NET assembly load events, with details such as the module's ID, assembly ID, application domain ID, assembly flags, and fully qualified assembly name. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6d5d39f9-3ec3-4c7d-bd65-6cece55767e3)

The **`AssemblyFlags`** indicate domain neutrality and whether the code is native or managed. The **`FullyQualifiedAssemblyName`** provides the name, version, culture, and public key token of the loaded assemblies, which are attributes used to uniquely identify .NET assemblies.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/341433fd-2ee3-4a42-b154-65653e7f66af)

# Detection

As discussed previously, .NET assemblies that are loaded reflectively do not typically reference a file path, since they are loaded directly from a byte array in memory. The event we're examining is coming from Event ID **152**. However, it's important to note that relying solely on this Event ID to determine if a .NET assembly is reflectively loaded could lead to false positives.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fa1513e4-c4f7-4f85-9de1-417bf4a61b61)


We could correlate the .NET assembly that is **loaded without a path** with Event ID **154** to provide additional context. The **`AssemblyFlags`** describes the characteristics of the .NET Assembly. When this flag is set to **`0`** for example, the assembly does not have special instructions for the runtime regarding loading and execution, such as being **DomainNeutral Native**, **Dynamic** or **Native**.

When the **`PublicKeyToken`** of a .NET assembly is set to **`null`**, it indicates that the assembly is not strongly named. Strong naming involves signing an assembly with a unique cryptographic public/private key pair. A null **`PublicKeyToken`** shows that this process has not been done.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/362f5d79-e8bd-46b0-89ca-60b6820caafd)

In this screenshot, which is from the CrowdStrike white paper, we observe indicators of a .NET assembly loaded reflectively. The **`ModuleILPath`** lacks a drive letter, the **`AssemblyFlags`** are set to **`0`**, and the **`FullyQualifiedAssemblyName`** field shows a **`PublicKeyToken`** of **`null`**, as well as the **`ModuleFlags`** is set to **`8`** (manifest), which are all consistent with reflective loading via byte arrays as discussed in the white paper.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/585d52d8-812f-4d56-9261-861a9ebcaf15)


CrowdStrike OverWatch team also explained in the whitepaper their strategies when they are building detections for reflective loaded .NET activities. This include the use of regular expressions to identify assembly names and monitoring for reflective .NET loads occurring under applications or IIS application pools where such loading behavior is unusual. Let's demonstrate an example to make it a bit more clear.

As we can see in this screenshot, it indicates that the IIS worker process has multiple .NET application domains with assemblies likely loaded reflectively from byte arrays, as identified by their naming pattern without traditional file paths. This would get logged in Event ID **152** of **`Microsoft-Windows-DotNETRuntime`** ETW provider, which we then could use the **`AssemblyID`** field to correlate it with Event ID **154** to determine whether the **`AssemblyFlags`** is set to **`0`**, and the **`FullyQualifiedAssemblyName`** field shows **`PublicKeyToken`** is set to **`null`**, which are all consistent with reflective loading via byte arrays.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0992b2e7-0fd2-4731-bf21-1bb94dd2255b)

The example mentioned above aligns with the description in the white paper. It details that the IIS Worker Process does not load its generated temporary .NET assemblies reflectively from byte arrays.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/461a0531-b0cd-4dbd-8dd0-28de925767da)


To summarize this in shortly on how EDR vendors could build detections based upon this available telemetry:

- .NET assemblies that starts with the prefix **App_Web_*** being reflectively loaded from byte array within the IIS Worker Process (w3wp.exe), where the **`AssemblyFlags`** is set to **`0`** and and the **`PublicKeyToken`** in the **`FullyQualifiedAssemblyName`** is set to **`null`**, as well as the **`ModuleFlags`** is set to **`8`** 
- .NET assemblies that are reflectively loaded from byte array in general where the **`AssemblyFlags`** is set to **`0`** and the **`PublicKeyToken`** in the **`FullyQualifiedAssemblyName`** is set to **`null`**, as well as the **`ModuleFlags`** is set to **`8`** 

It's also important to note that if we would just collect Event ID **154** and filter on **`AssemblyFlags`** set to **`0`** and the **`PublicKeyToken`** in the **`FullyQualifiedAssemblyName`** is set to **`null`**. It will generate a lot of noise, which is why the correlation needs to be made with **152** to reduce the noise. This allows us then to create better detections for .NET Assemblies being reflectively loaded.

# Fields

Here's a summary of the fields for Event ID **152**:

| Field           | Description |
|------------------------|-------------|
| ModuleID (Field 1)     | The identifier for the module associated with the event. |
| AssemblyID (Field 2)   | A unique identifier for the .NET assembly involved in the event. |
| ModuleFlags (Field 3)  | Flags providing additional information about the module. |
| ModuleILPath (Field 5) | The path to the Intermediate Language file for the module. |
| ModuleNativePath (Field 6) | The path to the native image file for the module. |
| ClrInstanceID (Field 7) | Identifies the instance of the CLR where the event occurred. |
| ManagedPdbSignature (Field 8) | The signature of the PDB for the managed code. |
| ManagedPdbAge (Field 9) | The age of the PDB for the managed code. |
| ManagedPdbBuildPath (Field 10) | The build path for the PDB of the managed code. |
| NativePdbSignature (Field 11) | The signature of the PDB for the native code. |
| NativePdbAge (Field 12) | The age of the PDB for the native code. |
| NativePdbBuildPath (Field 13) | The build path for the PDB of the native code. |

Here's a summary of the fields for Event ID **154**:

| Field              | Description |
|----------------------------|-------------|
| AssemblyID (Field 1)       | A unique identifier for the .NET assembly involved in the event. |
| AppDomainID (Field 2)      | The identifier for the application domain where the assembly is loaded. |
| AssemblyFlags (Field 4)    | Flags that describe characteristics of the assembly. |
| FullyQualifiedAssemblyName (Field 5) | The complete name of the assembly, including version, culture, and public key token. |
| ClrInstanceID (Field 6)    | Identifies the instance of the CLR where the event occurred. |

# Evasion

As we may know, this ETW Provider is very useful. However, it also has its downsides; one significant issue is that it can be disabled if someone passes **`COMPlus_ETWEnabled=0`** as an environment variable during a **`CreateProcess`** call for example. This has been documented here: https://blog.xpnsec.com/hiding-your-dotnet-complus-etwenabled/

Let's test this out: first, we will trace the **`Microsoft-Windows-DotNETRuntime`** ETW provider, and then we will start PowerShell. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/898237eb-bf9b-43ec-8f5e-78b64a7187d9)

Once PowerShell has started, we can just stop the tracing and analyze the ETW trace. This trace will display all the .NET assemblies loaded within the process, as shown in the following screenshot:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9285d73a-e2ca-4753-b4d2-1fc89b39a968)

I quickly looked online and found a proof of concept that lets you use **`COMPlus_ETWEnabled=0`** as an environment variable in a **`CreateProcess`** call to start PowerShell. https://gist.github.com/shantanu561993/0fff4e57a30e316068e0e9c8ac6d0eb2

Let's do the exact same thing again, which is starting to trace the ETW Provider and then execute the POC:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b6ba7fdd-56c7-4aae-bb49-f411ea5f5bbc)

As evident from this ETW trace, we are now missing events, and it no longer displays the .NET assembly loads compared to the previous screenshot. This serves to highlight that this ETW Provider can be relative easily be evaded.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/992e330c-cde5-4e79-a12e-249957677c35)

# Reference

- **IceApple: A Novel Internet Information Services (IIS) Post-Exploitation Framework:** https://www.crowdstrike.com/wp-content/uploads/2022/05/crowdstrike-iceapple-a-novel-internet-information-services-post-exploitation-framework-1.pdf
- **Hiding your .NET - COMPlus_ETWEnabled:** https://blog.xpnsec.com/hiding-your-dotnet-complus-etwenabled/
