# How to get started?

This section will cover all the necessary tools we will be using and how to set everything up. The following tools will be used:

- **Process Explorer**
- **Visual Studio Community**
- **Windows SDK**
- **WinDbg**
- **MEX Extension**

It is **recommended** to do everything on a virtual machine, instead of your personal machine.

# Process Explorer

First tool we will be downloading is Process Explorer. This is a tool that provides detailed information about running processes and system resource usage on a Windows operating system. We will be using the version **v17.05** or higher:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a094c17d-2c07-4b8f-ac49-b3f6114a8ab1)


**DOWNLOAD HERE:** https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer

# Visual Studio Community

The second tool we will be downloading is **Visual Studio Community**. Visual Studio Community is a free, fully-featured integrated development environment (IDE) from Microsoft, designed for coding and building various software applications.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0fa48737-3828-4f04-a043-837280f33ff7)


Since we will be working a lot with C/C++, make sure to select **Desktop Development with C++** during the installation of **Visual Studio Community**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/64210db7-ca3c-4c95-8b25-0356d95c7716)


Once Visual Studio Community has been installed. Let's create a first project together.

1. Click on **New** and then **Project**
2. Select **C++** and then **Console App**
3. Click **Next**



4. Provide a name and location and click on **Create**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/cceadee7-5b19-4f6e-8a7f-30b2e568b43d)

5. Right-click the project in the **Solution Explorer** and choose **Properties**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/90304ebe-4a33-4d82-87fb-f6873c514d6f)

6. Change the **Configurations** to **All Configurations** and **Platform** to **All Platforms**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/37c02d99-10e3-4385-b29e-6e697f3a30b6)

7. Go to **Configuration Properties** -> **C/C++** -> **Code Generation** -> **Runtime Library** -> Select **Multi-threaded /MT**

 ![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/15f4ed8e-35f5-4298-86c1-cf870765b722)

8. Save the project

**DOWNLOAD HERE:** https://visualstudio.microsoft.com/downloads/

# Windows SDK

The Windows SDK provides code and tools to aid the development and troubleshooting of applications and systems. Make sure to install **Debugging Tools for Window**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b8a57f66-50ea-4890-9bf9-4d8e9ec24a6b)


**DOWNLOAD HERE:** https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/

# WinDbg

WinDbg is a powerful debugging tool from Microsoft used for analyzing and diagnosing software issues in Windows applications and operating systems.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/07a00218-66b6-4725-a4e7-1a5b3f1157fe)

**DOWNLOAD HERE:** https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/

# How to configure Symbols?

Symbol files hold a variety of data which are not actually needed when running the binaries, but which could be very useful in the debugging process. Symbols can include the name, type (if applicable), the address or register where it is stored, and any parent or child symbols. Examples of symbols include variable names (local and global), functions, and any entry point into a module. 

1. Open **View advanced system settings**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b38ccc2d-7a9b-43a2-b0da-98b6625b78b0)

2. Click on **Environment Variables**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/8eec1bc5-80af-427b-b202-6197d8ebf1cc)

3. Configure the following environment variables:

| Variable Name      | Variable Value                                              |
|--------------------|-------------------------------------------------------------|
| `_NT_SYMBOL_PATH`  | `SRV*C:\Symbols*https://msdl.microsoft.com/download/symbols` |
| `_NT_SOURCE_PATH`  | `SRV*C:\Symbols*https://msdl.microsoft.com/download/symbols` |
| `_NT_SYMCACHE_PATH`| `C:\SymCache`                                               |


This is how the end result should look like:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/e997836d-0af7-422a-8919-843cd083cacf)

Open **Process Explorer** and go to **Options** and then configure the symbols:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/57b909bb-8ade-457f-ba0c-500f0a7bd71a)

# Console Debugger (CDB)

There are people that may prefer to use the Console Debugger above the WinDbg GUI itself. Since we have installed Windows SDK previously, we can setup this as well.

1. Open **View advanced system settings**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b38ccc2d-7a9b-43a2-b0da-98b6625b78b0)

2. Click on **Environment Variables**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/8eec1bc5-80af-427b-b202-6197d8ebf1cc)

3. Under **User variables for User** select **Path** and then paste the following location:

```
C:\Program Files (x86)\Windows Kits\10\Debuggers\x64
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/8afa5907-f29a-4725-8a8e-a73860edc9d1)


4. We can now use **cdb.exe** to analyze memory dumps for example:
   
```
cdb -z C:\Users\User\Desktop\MEMORY.DMP
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7214b515-4182-40eb-a77a-08e3bfc7932d)

# Install the MEX Extension

MEX Debugging Extension for WinDbg can help you simplify common debugger tasks, and provides powerful text filtering capabilities to the debugger.

1. Go to the following link and download **MEX**: https://www.microsoft.com/en-US/download/details.aspx?id=53304
2. Extract the ZIP file

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/0fc635e4-0f34-473d-8ce5-fbbbf8240e4d)

3. Launch **notepad.exe** as an example in WinDbg. It doesn't really matter which one.
4. Run the following command: **.extpath**

This will display a similar result like the following:

```
0:000> .extpath
Extension search path is: C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2306.14001.0_x64__8wekyb3d8bbwe\amd64\WINXP;C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2306.14001.0_x64__8wekyb3d8bbwe\amd64\winext;C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2306.14001.0_x64__8wekyb3d8bbwe\amd64\winext\arcade;C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2306.14001.0_x64__8wekyb3d8bbwe\amd64\pri;C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2306.14001.0_x64__8wekyb3d8bbwe\amd64;C:\Users\Admin\AppData\Local\Dbg\EngineExtensions;C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2306.14001.0_x64__8wekyb3d8bbwe\amd64;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files\Microsoft SQL Server\150\Tools\Binn\;C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\170\Tools\Binn\;C:\Program Files\dotnet\;C:\Program Files (x86)\Windows Kits\10\Windows Performance Toolkit\;C:\Users\Admin\AppData\Local\Microsoft\WindowsApps;C:\Users\Admin\.dotnet\tools;C:\Program Files (x86)\Windows Kits\10\Debuggers\x86
```

5. Pick one of the above locations and store the **mex.dll** extension. In this example, we are going to pick **C:\Users\Admin\AppData\Local\Dbg\EngineExtensionst**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4f04b025-8f37-48a8-9623-9a4f548a99b5)

6. When we now type **.load mex** in WinDbg, we will get to see:

```
1:004> .load mex
Mex External 3.0.0.7172 Loaded!
```
