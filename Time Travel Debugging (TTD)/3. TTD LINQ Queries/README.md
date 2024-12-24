# Summary

The debugger data model is a flexible object model that serves as the backbone for how modern debugger extensions interact with the debugger and share information. Using the data model APIs, this information becomes available through the debugger's modern `dx` command, as well as through extensions written in JavaScript or C++.

To understand why the data model is useful, consider this traditional debugger command:

```
0:000> lm
start    end        module name
00a90000 00c87000   Basta    C (no symbols)           
74c90000 74dfc000   TTDRecordCPU   (deferred)             
74e00000 75027000   COMCTL32   (deferred)             
75040000 75046000   PSAPI      (deferred)             
75110000 75189000   msvcp_win   (deferred)             
75230000 754b3000   KERNELBASE   (deferred)             
75520000 75632000   ucrtbase   (deferred)             
756d0000 75788000   COMDLG32   (deferred)

<<< SNIPPET >>>
```

This older command produces text-based output that’s difficult to parse, customize, or reuse in other contexts. The format is rigid and specific to the command. The debugger data model replaces this with a structured and extensible way to interact with the debugger.

```
0:000> dx @$curprocess.TTD.Events.Where(e => e.Type == "ModuleLoaded")
@$curprocess.TTD.Events.Where(e => e.Type == "ModuleLoaded")                
    [0x0]            : Module Basta.exe Loaded at position: 2:0
    [0x1]            : Module TTDRecordCPU.dll Loaded at position: 3:0
    [0x2]            : Module COMCTL32.dll Loaded at position: 4:0
    [0x3]            : Module PSAPI.DLL Loaded at position: 5:0
    [0x4]            : Module msvcp_win.dll Loaded at position: 6:0
    [0x5]            : Module KERNELBASE.dll Loaded at position: 7:0

<<< SNIPPET >>>
```

## Tracing sample

In this example, we captured a TTD trace of a Black Basta ransomware sample, and the next step is to analyze it in WinDbg using LINQ queries.

![image](https://github.com/user-attachments/assets/2f6c7dfd-7c0b-41be-9a74-a005a14b2575)


Open the trace file in WinDbg to begin the analysis.

![image](https://github.com/user-attachments/assets/0b11be3d-e14d-45e1-933b-7f953e69bf75)


## Example

In this example, we’re working with a trace file from a Black Basta ransomware sample. Let’s explore it using a few sample commands.


This command displays all the loaded modules, along with the point in the trace where each was loaded.

```
0:000> dx @$curprocess.TTD.Events.Where(e => e.Type == "ModuleLoaded")
@$curprocess.TTD.Events.Where(e => e.Type == "ModuleLoaded")                
    [0x0]            : Module Basta.exe Loaded at position: 2:0
    [0x1]            : Module TTDRecordCPU.dll Loaded at position: 3:0
    [0x2]            : Module COMCTL32.dll Loaded at position: 4:0
    [0x3]            : Module PSAPI.DLL Loaded at position: 5:0
    [0x4]            : Module msvcp_win.dll Loaded at position: 6:0
    [0x5]            : Module KERNELBASE.dll Loaded at position: 7:0
    [0x6]            : Module ucrtbase.dll Loaded at position: 8:0
    [0x7]            : Module COMDLG32.dll Loaded at position: 9:0
    [0x8]            : Module win32u.dll Loaded at position: A:0
    [0x9]            : Module gdi32full.dll Loaded at position: B:0
    [0xa]            : Module USER32.dll Loaded at position: C:0
    [0xb]            : Module IMM32.DLL Loaded at position: D:0
    [0xc]            : Module msvcrt.dll Loaded at position: E:0
    [0xd]            : Module KERNEL32.DLL Loaded at position: F:0
    [0xe]            : Module shcore.dll Loaded at position: 10:0
    [0xf]            : Module ADVAPI32.dll Loaded at position: 11:0
    [0x10]           : Module GDI32.dll Loaded at position: 12:0
    [0x11]           : Module bcrypt.dll Loaded at position: 13:0
    [0x12]           : Module ole32.dll Loaded at position: 14:0
    [0x13]           : Module SHLWAPI.dll Loaded at position: 15:0
    [0x14]           : Module SHELL32.dll Loaded at position: 16:0
    [0x15]           : Module RPCRT4.dll Loaded at position: 17:0
    [0x16]           : Module combase.dll Loaded at position: 18:0
    [0x17]           : Module sechost.dll Loaded at position: 19:0
    [0x18]           : Module ntdll.dll Loaded at position: 1A:0
    [0x19]           : Module UxTheme.dll Loaded at position: 9E:0
    [0x1a]           : Module wintypes.dll Loaded at position: 1F3:0
    [0x1b]           : Module windows.storage.dll Loaded at position: 1F8:0
    [0x1c]           : Module profapi.dll Loaded at position: 3BC:0
    [0x1d]           : Module kernel.appcore.dll Loaded at position: 5E5:0
    [0x1e]           : Module bcryptPrimitives.dll Loaded at position: 619:0
    [0x1f]           : Module OLEAUT32.dll Loaded at position: 1BD26D:0
    [0x20]           : Module WLDAP32.dll Loaded at position: 1BD2B8:0
    [0x21]           : Module CRYPTSP.dll Loaded at position: 1BD3C8:0
    [0x22]           : Module rsaenh.dll Loaded at position: 1BD42C:0
    [0x23]           : Module CRYPTBASE.dll Loaded at position: 1BD49F:0
```

The following command shows the Windows APIs that were imported, but it doesn’t guarantee that the APIs were actually called.

```
0:000> dx -r2 @$curprocess.Modules[0].Contents.Imports.Select(m => m.Functions)
@$curprocess.Modules[0].Contents.Imports.Select(m => m.Functions)                
    ["SHLWAPI.dll"] 
        [0x0]            : Named Import of 'PathGetDriveNumberW'
        [0x1]            : Named Import of 'StrCmpNIW'
        [0x2]            : Named Import of 'StrDupW'
        [0x3]            : Named Import of 'StrChrA'
        [0x4]            : Named Import of 'PathRelativePathToW'
        [0x5]            : Named Import of 'PathIsPrefixW'
        [0x6]            : Named Import of 'PathFindFileNameW'
        [0x7]            : Named Import of 'PathUnExpandEnvStringsW'
        [0x8]            : Named Import of 'PathIsRootW'
        [0x9]            : Named Import of 'PathCanonicalizeW'
        [0xa]            : Named Import of 'PathFindExtensionW'
        [0xb]            : Named Import of 'PathCommonPrefixW'
        [0xc]            : Named Import of 'PathCompactPathExW'
        [0xd]            : Named Import of 'PathRemoveExtensionW'
        [0xe]            : Named Import of 'StrFormatByteSizeW'
        [0xf]            : Named Import of 'PathStripPathW'
        [0x10]           : Named Import of 'PathRemoveBackslashW'
        [0x11]           : Named Import of 'StrRetToBufW'
        [0x12]           : Named Import of 'PathMatchSpecW'
        [0x13]           : Named Import of 'StrCatBuffW'
        [0x14]           : Named Import of 'PathUnquoteSpacesW'
        [0x15]           : Named Import of 'StrChrW'
        [0x16]           : Named Import of 'StrTrimW'
        [0x17]           : Named Import of 'SHAutoComplete'
        [0x18]           : Named Import of 'StrCpyNW'
        [0x19]           : Named Import of 'PathQuoteSpacesW'
        [0x1a]           : Named Import of 'PathRenameExtensionW'
        [0x1b]           : Named Import of 'PathIsDirectoryW'
        [0x1c]           : Named Import of 'StrRChrW'
        [0x1d]           : Named Import of 'PathAppendW'
        [0x1e]           : Named Import of 'PathIsRelativeW'
        [0x1f]           : Named Import of 'PathFileExistsW'
        [0x20]           : Named Import of 'PathAddBackslashW'
        [0x21]           : Named Import of 'PathRemoveFileSpecW'
        [0x22]           : Named Import of 'PathIsSameRootW'
    ["PSAPI.DLL"]   
        [0x0]            : Named Import of 'EnumProcessModules'
        [0x1]            : Named Import of 'GetModuleFileNameExW'
    ["USER32.dll"]  
        [0x0]            : Named Import of 'OffsetRect'
        [0x1]            : Named Import of 'OpenClipboard'
        [0x2]            : Named Import of 'BeginDeferWindowPos'
        [0x3]            : Named Import of 'GetSubMenu'
        [0x4]            : Named Import of 'TrackPopupMenu'
        [0x5]            : Named Import of 'LoadAcceleratorsW'
        [0x6]            : Named Import of 'DeleteMenu'
        [0x7]            : Named Import of 'ShowOwnedPopups'
        [0x8]            : Named Import of 'CopyImage'
        [0x9]            : Named Import of 'MessageBoxW'
        [0xa]            : Named Import of 'EqualRect'
        [0xb]            : Named Import of 'IsWindowVisible'
        [0xc]            : Named Import of 'ShowWindowAsync'
        [0xd]            : Named Import of 'GetMessagePos'
        [0xe]            : Named Import of 'LoadMenuW'
        [0xf]            : Named Import of 'CharUpperW'
        [0x10]           : Named Import of 'GetKeyState'
        [0x11]           : Named Import of 'DefWindowProcW'
        [0x12]           : Named Import of 'GetMenuItemInfoW'
        [0x13]           : Named Import of 'DeferWindowPos'
        [0x14]           : Named Import of 'GetMessageW'
        [0x15]           : Named Import of 'CloseClipboard'
        [0x16]           : Named Import of 'SetMenuItemInfoW'
        [0x17]           : Named Import of 'EmptyClipboard'
        [0x18]           : Named Import of 'RegisterClassW'
        [0x19]           : Named Import of 'SetWindowPlacement'
        [0x1a]           : Named Import of 'FrameRect'
        [0x1b]           : Named Import of 'SetMenuDefaultItem'
        [0x1c]           : Named Import of 'EnumWindows'
        [0x1d]           : Named Import of 'GetMessageTime'
        [0x1e]           : Named Import of 'IntersectRect'
        [0x1f]           : Named Import of 'SetFocus'
        [0x20]           : Named Import of 'BringWindowToTop'
        [0x21]           : Named Import of 'TranslateAcceleratorW'
        [0x22]           : Named Import of 'GetWindowDC'
        [0x23]           : Named Import of 'EndDeferWindowPos'
        [0x24]           : Named Import of 'SetClipboardData'
        [0x25]           : Named Import of 'CheckMenuItem'
        [0x26]           : Named Import of 'IsZoomed'
        [0x27]           : Named Import of 'KillTimer'
        [0x28]           : Named Import of 'PostQuitMessage'
        [0x29]           : Named Import of 'GetSysColorBrush'
        [0x2a]           : Named Import of 'EnableMenuItem'
        [0x2b]           : Named Import of 'RegisterWindowMessageW'
        [0x2c]           : Named Import of 'UpdateWindow'
        [0x2d]           : Named Import of 'IsIconic'
        [0x2e]           : Named Import of 'GetWindowThreadProcessId'
        [0x2f]           : Named Import of 'DrawAnimatedRects'
        [0x30]           : Named Import of 'FindWindowExW'
        [0x31]           : Named Import of 'GetDC'
        [0x32]           : Named Import of 'MonitorFromRect'
        [0x33]           : Named Import of 'SetActiveWindow'
        [0x34]           : Named Import of 'LoadStringA'
        [0x35]           : Named Import of 'SetWindowTextW'
        [0x36]           : Named Import of 'LoadStringW'
        [0x37]           : Named Import of 'DdeCreateStringHandleW'
        [0x38]           : Named Import of 'DdeConnect'
        [0x39]           : Named Import of 'GetMonitorInfoW'
        [0x3a]           : Named Import of 'DdeInitializeW'
        [0x3b]           : Named Import of 'SetTimer'
        [0x3c]           : Named Import of 'SetWindowCompositionAttribute'
        [0x3d]           : Named Import of 'SystemParametersInfoW'
        [0x3e]           : Named Import of 'SetPropW'
        [0x3f]           : Named Import of 'RedrawWindow'
        [0x40]           : Named Import of 'SendMessageW'
        [0x41]           : Named Import of 'wsprintfW'
        [0x42]           : Named Import of 'GetSysColor'
        [0x43]           : Named Import of 'CharPrevW'
        [0x44]           : Named Import of 'GetWindowPlacement'
        [0x45]           : Named Import of 'GetSystemMetrics'
        [0x46]           : Named Import of 'DdeUninitialize'
        [0x47]           : Named Import of 'DialogBoxIndirectParamW'
        [0x48]           : Named Import of 'DdeClientTransaction'
        [0x49]           : Named Import of 'SetLayeredWindowAttributes'
        [0x4a]           : Named Import of 'CharUpperBuffW'
        [0x4b]           : Named Import of 'SetRect'
        [0x4c]           : Named Import of 'DdeDisconnect'
        [0x4d]           : Named Import of 'SetForegroundWindow'
        [0x4e]           : Named Import of 'LoadImageW'
        [0x4f]           : Named Import of 'ReleaseDC'
        [0x50]           : Named Import of 'GetPropW'
        [0x51]           : Named Import of 'RemovePropW'
        [0x52]           : Named Import of 'DispatchMessageW'
        [0x53]           : Named Import of 'PeekMessageW'
        [0x54]           : Named Import of 'TranslateMessage'
        [0x55]           : Named Import of 'GetWindowLongW'
        [0x56]           : Named Import of 'GetWindowTextLengthW'
        [0x57]           : Named Import of 'GetSystemMenu'
        [0x58]           : Named Import of 'AdjustWindowRectEx'
        [0x59]           : Named Import of 'PostMessageW'
        [0x5a]           : Named Import of 'CheckMenuRadioItem'
        [0x5b]           : Named Import of 'GetWindowRect'
        [0x5c]           : Named Import of 'GetFocus'
        [0x5d]           : Named Import of 'DestroyWindow'
        [0x5e]           : Named Import of 'SetWindowPos'
        [0x5f]           : Named Import of 'CheckRadioButton'
        [0x60]           : Named Import of 'MessageBoxExW'
        [0x61]           : Named Import of 'CreateWindowExW'
        [0x62]           : Named Import of 'EndDialog'
        [0x63]           : Named Import of 'MessageBeep'
        [...]           
    ["KERNEL32.dll"]
        [0x0]            : Named Import of 'RaiseException'
        [0x1]            : Named Import of 'GetSystemInfo'
        [0x2]            : Named Import of 'VirtualQuery'
        [0x3]            : Named Import of 'GetModuleHandleW'
        [0x4]            : Named Import of 'LoadLibraryExA'
        [0x5]            : Named Import of 'EnterCriticalSection'
        [0x6]            : Named Import of 'LeaveCriticalSection'
        [0x7]            : Named Import of 'DecodePointer'
        [0x8]            : Named Import of 'InitializeCriticalSectionAndSpinCount'
        [0x9]            : Named Import of 'DeleteCriticalSection'
        [0xa]            : Named Import of 'WaitForSingleObjectEx'
        [0xb]            : Named Import of 'ReadConsoleW'
        [0xc]            : Named Import of 'GetConsoleMode'
        [0xd]            : Named Import of 'VirtualProtect'
        [0xe]            : Named Import of 'CompareStringOrdinal'
        [0xf]            : Named Import of 'FreeLibrary'
        [0x10]           : Named Import of 'LoadLibraryExW'
        [0x11]           : Named Import of 'ReadFile'
        [0x12]           : Named Import of 'lstrlenW'
        [0x13]           : Named Import of 'WriteFile'
        [0x14]           : Named Import of 'lstrcpynW'
        [0x15]           : Named Import of 'ExpandEnvironmentStringsW'
        [0x16]           : Named Import of 'GetModuleFileNameW'
        [0x17]           : Named Import of 'SetFilePointer'
        [0x18]           : Named Import of 'SetEndOfFile'
        [0x19]           : Named Import of 'UnlockFileEx'
        [0x1a]           : Named Import of 'CreateFileW'
        [0x1b]           : Named Import of 'GetSystemDirectoryW'
        [0x1c]           : Named Import of 'MultiByteToWideChar'
        [0x1d]           : Named Import of 'lstrcatW'
        [0x1e]           : Named Import of 'CloseHandle'
        [0x1f]           : Named Import of 'LockFileEx'
        [0x20]           : Named Import of 'GetFileSize'
        [0x21]           : Named Import of 'WideCharToMultiByte'
        [0x22]           : Named Import of 'lstrcpyW'
        [0x23]           : Named Import of 'lstrcmpiW'
        [0x24]           : Named Import of 'lstrcmpW'
        [0x25]           : Named Import of 'FlushFileBuffers'
        [0x26]           : Named Import of 'GetShortPathNameW'
        [0x27]           : Named Import of 'LocalAlloc'
        [0x28]           : Named Import of 'GetFileAttributesW'
        [0x29]           : Named Import of 'SetFileAttributesW'
        [0x2a]           : Named Import of 'FormatMessageW'
        [0x2b]           : Named Import of 'GetLastError'
        [0x2c]           : Named Import of 'GetCurrentDirectoryW'
        [0x2d]           : Named Import of 'LocalFree'
        [0x2e]           : Named Import of 'WaitForSingleObject'
        [0x2f]           : Named Import of 'CreateEventW'
        [0x30]           : Named Import of 'SetEvent'
        [0x31]           : Named Import of 'GlobalAlloc'
        [0x32]           : Named Import of 'GlobalFree'
        [0x33]           : Named Import of 'ResetEvent'
        [0x34]           : Named Import of 'SizeofResource'
        [0x35]           : Named Import of 'SearchPathW'
        [0x36]           : Named Import of 'GetLocaleInfoEx'
        [0x37]           : Named Import of 'FreeResource'
        [0x38]           : Named Import of 'OpenProcess'
        [0x39]           : Named Import of 'LockResource'
        [0x3a]           : Named Import of 'LoadLibraryW'
        [0x3b]           : Named Import of 'LoadResource'
        [0x3c]           : Named Import of 'FindResourceW'
        [0x3d]           : Named Import of 'GetWindowsDirectoryW'
        [0x3e]           : Named Import of 'GetProcAddress'
        [0x3f]           : Named Import of 'GlobalLock'
        [0x40]           : Named Import of 'GlobalUnlock'
        [0x41]           : Named Import of 'MulDiv'
        [0x42]           : Named Import of 'CreateDirectoryW'
        [0x43]           : Named Import of 'FindFirstFileW'
        [0x44]           : Named Import of 'GetCommandLineW'
        [0x45]           : Named Import of 'SetErrorMode'
        [0x46]           : Named Import of 'FindClose'
        [0x47]           : Named Import of 'GetUserPreferredUILanguages'
        [0x48]           : Named Import of 'FindFirstChangeNotificationW'
        [0x49]           : Named Import of 'GetVersion'
        [0x4a]           : Named Import of 'ResolveLocaleName'
        [0x4b]           : Named Import of 'GlobalSize'
        [0x4c]           : Named Import of 'FileTimeToSystemTime'
        [0x4d]           : Named Import of 'FindCloseChangeNotification'
        [0x4e]           : Named Import of 'FileTimeToLocalFileTime'
        [0x4f]           : Named Import of 'FindNextChangeNotification'
        [0x50]           : Named Import of 'SetCurrentDirectoryW'
        [0x51]           : Named Import of 'GetTimeFormatW'
        [0x52]           : Named Import of 'ExitProcess'
        [0x53]           : Named Import of 'VerSetConditionMask'
        [0x54]           : Named Import of 'CopyFileW'
        [0x55]           : Named Import of 'VerifyVersionInfoW'
        [0x56]           : Named Import of 'GetDateFormatW'
        [0x57]           : Named Import of 'MapViewOfFile'
        [0x58]           : Named Import of 'CreateFileMappingW'
        [0x59]           : Named Import of 'LocaleNameToLCID'
        [0x5a]           : Named Import of 'FindResourceExW'
        [0x5b]           : Named Import of 'LCIDToLocaleName'
        [0x5c]           : Named Import of 'UnmapViewOfFile'
        [0x5d]           : Named Import of 'GetVersionExW'
        [0x5e]           : Named Import of 'GetLocaleInfoW'
        [0x5f]           : Named Import of 'GetUserDefaultUILanguage'
        [0x60]           : Named Import of 'GetSystemDefaultUILanguage'
        [0x61]           : Named Import of 'SetLastError'
        [0x62]           : Named Import of 'UnhandledExceptionFilter'
        [0x63]           : Named Import of 'GetConsoleOutputCP'
        [...]           
    ["GDI32.dll"]   
        [0x0]            : Named Import of 'GetStockObject'
        [0x1]            : Named Import of 'SetBkColor'
        [0x2]            : Named Import of 'ExtTextOutW'
        [0x3]            : Named Import of 'EnumFontsW'
        [0x4]            : Named Import of 'GetDeviceCaps'
        [0x5]            : Named Import of 'SetTextColor'
        [0x6]            : Named Import of 'GetObjectW'
        [0x7]            : Named Import of 'DeleteObject'
        [0x8]            : Named Import of 'CreateSolidBrush'
        [0x9]            : Named Import of 'CreateFontIndirectW'
    ["COMDLG32.dll"]
        [0x0]            : Named Import of 'GetSaveFileNameW'
        [0x1]            : Named Import of 'ChooseColorW'
        [0x2]            : Named Import of 'GetOpenFileNameW'
    ["ADVAPI32.dll"]
        [0x0]            : Named Import of 'RegOpenKeyExW'
        [0x1]            : Named Import of 'RegQueryValueExW'
        [0x2]            : Named Import of 'RegCloseKey'
    ["SHELL32.dll"] 
        [0x0]            : Named Import of 'SHGetFolderPathW'
        [0x1]            : Named Import of 'SHGetSpecialFolderPathW'
        [0x2]            : Named Import of 'ShellExecuteW'
        [0x3]            : Named Import of 'SHCreateDirectoryExW'
        [0x4]            : Named Import of 'SHFileOperationW'
        [0x5]            : Named Import of 'SHBrowseForFolderW'
        [0x6]            : Named Import of 'SHGetSpecialFolderLocation'
        [0x7]            : Named Import of 'ShellExecuteExW'
        [0x8]            : Named Import of 'SHGetPathFromIDListW'
        [0x9]            : Named Import of 'SHGetFileInfoW'
        [0xa]            : Named Import of 'SHGetDesktopFolder'
        [0xb]            : Ordinal Import of #2147483828
        [0xc]            : Named Import of 'SHAppBarMessage'
        [0xd]            : Named Import of 'DragQueryFileW'
        [0xe]            : Named Import of 'Shell_NotifyIconW'
        [0xf]            : Named Import of 'DragAcceptFiles'
        [0x10]           : Named Import of 'DragFinish'
        [0x11]           : Named Import of 'SHGetDataFromIDListW'
    ["ole32.dll"]   
        [0x0]            : Named Import of 'OleUninitialize'
        [0x1]            : Named Import of 'CoCreateInstance'
        [0x2]            : Named Import of 'OleInitialize'
        [0x3]            : Named Import of 'CoUninitialize'
        [0x4]            : Named Import of 'CoTaskMemAlloc'
        [0x5]            : Named Import of 'CoTaskMemFree'
        [0x6]            : Named Import of 'CoInitialize'
        [0x7]            : Named Import of 'DoDragDrop'
    ["ntdll.dll"]   
        [0x0]            : Named Import of 'RtlGetNtVersionNumbers'
    ["COMCTL32.dll"]
        [0x0]            : Named Import of 'ImageList_AddMasked'
        [0x1]            : Named Import of 'InitCommonControlsEx'
        [0x2]            : Ordinal Import of #2147484058
        [0x3]            : Ordinal Import of #2147484061
        [0x4]            : Named Import of 'ImageList_Create'
        [0x5]            : Named Import of 'ImageList_Destroy'
        [0x6]            : Ordinal Import of #2147484029
        [0x7]            : Named Import of 'PropertySheetW'
```

To pinpoint the specific Windows APIs called from a particular module, you can use a command like this:

```
0:000> dx @$cursession.TTD.Calls("shell32!*").Select(c => c.Function).Distinct()
@$cursession.TTD.Calls("shell32!*").Select(c => c.Function).Distinct()                
    [0x0]            : SHELL32!_DllMainCRTStartup
    [0x1]            : SHELL32!dllmain_dispatch
    [0x2]            : SHELL32!_SEH_prolog4
    [0x3]            : SHELL32!dllmain_raw
    [0x4]            : SHELL32!dllmain_crt_dispatch
    [0x5]            : SHELL32!__scrt_dllmain_crt_thread_attach
    [0x6]            : SHELL32!wistd::__function::__func<<lambda_3e52fec36ab1c34d94fe5e5f93735475>,bool
    [0x7]            : SHELL32!DllMain
    [0x8]            : SHELL32!ATL_DllMain
    [0x9]            : SHELL32!SHGetFolderPathWStub
    [0xa]            : SHELL32!_imp_load__SHGetFolderPathW
    [0xb]            : SHELL32!_tailMerge_api_ms_win_shell_shellfolders_l1_1_0_dll
    [0xc]            : SHELL32!__delayLoadHelper2
    [0xd]            : SHELL32!SHGetSpecialFolderPathWStub
    [0xe]            : SHELL32!SHChangeNotifyStub
    [0xf]            : SHELL32!_imp_load__SHChangeNotify
    [0x10]           : SHELL32!_tailMerge_api_ms_win_shell_changenotify_l1_1_0_dll
```

Let’s check how many times `CreateProcessW` was called.

```
0:000> dx @$cursession.TTD.Calls("kernelbase!CreateProcessW").Count()
@$cursession.TTD.Calls("kernelbase!CreateProcessW").Count() : 0x1
```

In this example, `CreateProcessW` was called once. Next, let’s navigate to the point in the trace where `CreateProcessW` was invoked.

```
0:000> dx @$cursession.TTD.Calls("kernelbase!CreateProcessW").First().TimeStart.SeekTo()
(1008.1498): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 1BD50E:26
@$cursession.TTD.Calls("kernelbase!CreateProcessW").First().TimeStart.SeekTo()
```

This output shows the current CPU register state and the instruction being executed. The debugger has paused at the `CreateProcessW` function in `KERNELBASE.dll`.

```
0:000> r
eax=00000000 ebx=00000000 ecx=d97bcec6 edx=014856cc esi=00000000 edi=00000044
eip=7536a820 esp=00a3f96c ebp=00a3f9e8 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
KERNELBASE!CreateProcessW:
7536a820 8bff            mov     edi,edi
```

Now that we’re at the point where `CreateProcessW` is invoked, you can use the `!pde.dpx -du` command to display various types of information, such as the stack or memory around specific addresses. The `-du` is used to display Unicode strings. The output indicates that volume shadow copies are being deleted.

```
0:000> !pde.dpx -du
Start memory scan  : 0x00a3f96c ($csp)
End memory scan    : 0x00a40000 (User Stack Base)

0x00a3f970 : 0x01446400 :  !du "C:\Windows\system32\cmd.exe"
0x00a3f974 : 0x014476f0 :  !du ""C:\Windows\system32\cmd.exe /c C:\Windows\SysNative\vssadmin.exe delete shadows ...""
0x00a3f9a8 : 0x01446400 :  !du "C:\Windows\system32\cmd.exe"
0x00a3f9c0 : 0x014476f0 :  !du ""C:\Windows\system32\cmd.exe /c C:\Windows\SysNative\vssadmin.exe delete shadows ...""
0x00a3fa78 : 0x01446400 :  !du "C:\Windows\system32\cmd.exe"
0x00a3fa84 : 0x01446400 :  !du "C:\Windows\system32\cmd.exe"
0x00a3fb18 : 0x014358dc :  !du "C:\Users\WDAGUtilityAccount\Desktop\Basta.exe"
0x00a3fb1c : 0x018b1b88 : 0x01893070 :  !du ""         (((((                  H""
0x00a3fc10 : 0x00b1df98 :  !du "MiniPath"
```

The `!position` command displays your current location in the trace, which in this case is where `CreateProcessW` was invoked. You can run this command and note the position in the "Notes" section, for example.

```
0:000> !position
Time Travel Position: 1BD50E:26
```

![image](https://github.com/user-attachments/assets/952dc5f4-0b56-44d7-97c5-e5c1eb241998)


`TimeStart` indicates the position of a call within the trace and can be used to navigate between different points. `SystemTimeStart` represents the actual clock time of the call.

```
0:000> dx -g @$cursession.TTD.Calls("kernelbase!CreateFileW*").Select(c => new { Function = c.Function, TimeStart = c.TimeStart, SystemTimeStart = c.SystemTimeStart })
========================================================================================================
=           = (+) Function              = (+) TimeStart  = (+) SystemTimeStart                         =
========================================================================================================
= [0x0]     - KERNELBASE!CreateFileW    - 1BD570:5A3     - Thursday, November 21, 2024 11:06:11.82     =
= [0x1]     - KERNELBASE!CreateFileW    - 1BD694:23E8    - Thursday, November 21, 2024 11:06:11.114    =
= [0x2]     - KERNELBASE!CreateFileW    - 1BD69D:7DE     - Thursday, November 21, 2024 11:06:11.114    =
= [0x3]     - KERNELBASE!CreateFileW    - 1BDB6E:160     - Thursday, November 21, 2024 11:06:11.114    =
= [0x4]     - KERNELBASE!CreateFileW    - 1BE230:E31     - Thursday, November 21, 2024 11:06:11.177    =
= [0x5]     - KERNELBASE!CreateFileW    - 1BE2C3:B6B     - Thursday, November 21, 2024 11:06:11.177    =
= [0x6]     - KERNELBASE!CreateFileW    - 1BE34C:D72     - Thursday, November 21, 2024 11:06:11.208    =
= [0x7]     - KERNELBASE!CreateFileW    - 1BE5D1:B3F     - Thursday, November 21, 2024 11:06:11.271    =
= [0x8]     - KERNELBASE!CreateFileW    - 1BE74A:6B      - Thursday, November 21, 2024 11:06:11.333    =
= [0x9]     - KERNELBASE!CreateFileW    - 1BE7E4:B57     - Thursday, November 21, 2024 11:06:11.348    =
= [0xa]     - KERNELBASE!CreateFileW    - 1BE8B3:B57     - Thursday, November 21, 2024 11:06:11.364    =
= [0xb]     - KERNELBASE!CreateFileW    - 1BE90E:B45     - Thursday, November 21, 2024 11:06:11.380    =
= [0xc]     - KERNELBASE!CreateFileW    - 1BEDC1:26      - Thursday, November 21, 2024 11:06:11.396    =
= [0xd]     - KERNELBASE!CreateFileW    - 1BEF24:995     - Thursday, November 21, 2024 11:06:11.411    =
= [0xe]     - KERNELBASE!CreateFileW    - 1BF0AB:86D     - Thursday, November 21, 2024 11:06:11.411    =
= [0xf]     - KERNELBASE!CreateFileW    - 1BF40E:13E     - Thursday, November 21, 2024 11:06:11.458    =
= [0x10]    - KERNELBASE!CreateFileW    - 1BF4D5:406     - Thursday, November 21, 2024 11:06:11.458    =
= [0x11]    - KERNELBASE!CreateFileW    - 1BF542:6B      - Thursday, November 21, 2024 11:06:11.458    =
= [0x12]    - KERNELBASE!CreateFileW    - 1BFF33:26      - Thursday, November 21, 2024 11:06:11.490    =
= [0x13]    - KERNELBASE!CreateFileW    - 1C01A0:26      - Thursday, November 21, 2024 11:06:11.490    =
= [0x14]    - KERNELBASE!CreateFileW    - 1C030C:26      - Thursday, November 21, 2024 11:06:11.490    =
= [0x15]    - KERNELBASE!CreateFileW    - 1C0F6A:26      - Thursday, November 21, 2024 11:06:11.490    =
= [0x16]    - KERNELBASE!CreateFileW    - 1C1456:26      - Thursday, November 21, 2024 11:06:11.521    =
= [0x17]    - KERNELBASE!CreateFileW    - 1C14AA:26      - Thursday, November 21, 2024 11:06:11.505    =
= [0x18]    - KERNELBASE!CreateFileW    - 1C14E1:26      - Thursday, November 21, 2024 11:06:11.505    =
= [0x19]    - KERNELBASE!CreateFileW    - 1C2404:26      - Thursday, November 21, 2024 11:06:11.521    =
= [0x1a]    - KERNELBASE!CreateFileW    - 1C2406:26      - Thursday, November 21, 2024 11:06:11.521    =
= [0x1b]    - KERNELBASE!CreateFileW    - 1C26AC:26      - Thursday, November 21, 2024 11:06:11.521    =
= [0x1c]    - KERNELBASE!CreateFileW    - 1C32DF:6B      - Thursday, November 21, 2024 11:06:11.567    =
= [0x1d]    - KERNELBASE!CreateFileW    - 1C34FD:26      - Thursday, November 21, 2024 11:06:11.552    =
= [0x1e]    - KERNELBASE!CreateFileW    - 1C39B4:26      - Thursday, November 21, 2024 11:06:11.552    =
= [0x1f]    - KERNELBASE!CreateFileW    - 1C3E98:26      - Thursday, November 21, 2024 11:06:11.552    =
= [0x20]    - KERNELBASE!CreateFileW    - 1C4799:26      - Thursday, November 21, 2024 11:06:11.552    =
= [0x21]    - KERNELBASE!CreateFileW    - 1CB480:26A     - Thursday, November 21, 2024 11:06:11.942    =
= [0x22]    - KERNELBASE!CreateFileW    - 1D0B26:FA      - Thursday, November 21, 2024 11:06:12.177    =
= [0x23]    - KERNELBASE!CreateFileW    - 1D1413:6B      - Thursday, November 21, 2024 11:06:12.208    =
= [0x24]    - KERNELBASE!CreateFileW    - 1DC810:6B      - Thursday, November 21, 2024 11:06:12.567    =
= [0x25]    - KERNELBASE!CreateFileW    - 1E53DA:6B      - Thursday, November 21, 2024 11:06:12.802    =
= [0x26]    - KERNELBASE!CreateFileW    - 1EF137:6B      - Thursday, November 21, 2024 11:06:12.911    =
= [0x27]    - KERNELBASE!CreateFileW    - 1F8737:6B      - Thursday, November 21, 2024 11:06:13.396    =
= [0x28]    - KERNELBASE!CreateFileW    - 1FCE81:6B      - Thursday, November 21, 2024 11:06:13.568    =
= [0x29]    - KERNELBASE!CreateFileW    - 204B0D:6B      - Thursday, November 21, 2024 11:06:13.833    =
= [0x2a]    - KERNELBASE!CreateFileW    - 213D7D:13E     - Thursday, November 21, 2024 11:06:14.5      =
= [0x2b]    - KERNELBASE!CreateFileW    - 213F1F:150     - Thursday, November 21, 2024 11:06:14.99     =
= [0x2c]    - KERNELBASE!CreateFileW    - 2150D1:6B      - Thursday, November 21, 2024 11:06:14.99     =
= [0x2d]    - KERNELBASE!CreateFileW    - 216DC6:138     - Thursday, November 21, 2024 11:06:14.130    =
= [0x2e]    - KERNELBASE!CreateFileW    - 2180A1:6B      - Thursday, November 21, 2024 11:06:14.146    =
= [0x2f]    - KERNELBASE!CreateFileW    - 21BEC0:14A     - Thursday, November 21, 2024 11:06:14.5      =
= [0x30]    - KERNELBASE!CreateFileW    - 21C1C3:138     - Thursday, November 21, 2024 11:06:14.192    =
= [0x31]    - KERNELBASE!CreateFileW    - 21D871:26      - Thursday, November 21, 2024 11:06:14.192    =
= [0x32]    - KERNELBASE!CreateFileW    - 21E2A0:26      - Thursday, November 21, 2024 11:06:14.192    =
= [0x33]    - KERNELBASE!CreateFileW    - 21E382:150     - Thursday, November 21, 2024 11:06:14.161    =
= [0x34]    - KERNELBASE!CreateFileW    - 21FAAF:26      - Thursday, November 21, 2024 11:06:14.161    =
= [0x35]    - KERNELBASE!CreateFileW    - 220D3E:26      - Thursday, November 21, 2024 11:06:14.161    =
= [0x36]    - KERNELBASE!CreateFileW    - 222003:14A     - Thursday, November 21, 2024 11:06:14.5      =
= [0x37]    - KERNELBASE!CreateFileW    - 222428:26      - Thursday, November 21, 2024 11:06:14.240    =
= [0x38]    - KERNELBASE!CreateFileW    - 2228F0:26      - Thursday, November 21, 2024 11:06:14.240    =
= [0x39]    - KERNELBASE!CreateFileW    - 22357E:26      - Thursday, November 21, 2024 11:06:14.255    =
= [0x3a]    - KERNELBASE!CreateFileW    - 228503:6B      - Thursday, November 21, 2024 11:06:14.192    =
= [0x3b]    - KERNELBASE!CreateFileW    - 23BD69:6B      - Thursday, November 21, 2024 11:06:14.192    =
= [0x3c]    - KERNELBASE!CreateFileW    - 23E269:26      - Thursday, November 21, 2024 11:06:14.552    =
= [0x3d]    - KERNELBASE!CreateFileW    - 23E972:26      - Thursday, November 21, 2024 11:06:14.567    =
= [0x3e]    - KERNELBASE!CreateFileW    - 2444B0:26      - Thursday, November 21, 2024 11:06:14.646    =
= [0x3f]    - KERNELBASE!CreateFileW    - 246BD4:26      - Thursday, November 21, 2024 11:06:14.677    =
= [0x40]    - KERNELBASE!CreateFileW    - 247450:6B      - Thursday, November 21, 2024 11:06:14.692    =
= [0x41]    - KERNELBASE!CreateFileW    - 24F1D9:26      - Thursday, November 21, 2024 11:06:14.802    =
= [0x42]    - KERNELBASE!CreateFileW    - 2513DA:26      - Thursday, November 21, 2024 11:06:14.833    =
= [0x43]    - KERNELBASE!CreateFileW    - 2535C5:26      - Thursday, November 21, 2024 11:06:14.864    =
= [0x44]    - KERNELBASE!CreateFileW    - 25A5D6:6B      - Thursday, November 21, 2024 11:06:14.990    =
= [0x45]    - KERNELBASE!CreateFileW    - 25DC08:26      - Thursday, November 21, 2024 11:06:15.52     =
= [0x46]    - KERNELBASE!CreateFileW    - 25E5D2:26      - Thursday, November 21, 2024 11:06:15.52     =
= [0x47]    - KERNELBASE!CreateFileW    - 26174D:6B      - Thursday, November 21, 2024 11:06:15.145    =
= [0x48]    - KERNELBASE!CreateFileW    - 2632FF:26      - Thursday, November 21, 2024 11:06:15.177    =
= [0x49]    - KERNELBASE!CreateFileW    - 263607:26      - Thursday, November 21, 2024 11:06:15.177    =
= [0x4a]    - KERNELBASE!CreateFileW    - 265A26:6B      - Thursday, November 21, 2024 11:06:15.349    =
= [0x4b]    - KERNELBASE!CreateFileW    - 26A382:6B      - Thursday, November 21, 2024 11:06:15.567    =
= [0x4c]    - KERNELBASE!CreateFileW    - 271728:257     - Thursday, November 21, 2024 11:06:15.849    =
= [0x4d]    - KERNELBASE!CreateFileW    - 275A4B:6B      - Thursday, November 21, 2024 11:06:16.52     =
= [0x4e]    - KERNELBASE!CreateFileW    - 27EE87:6B      - Thursday, November 21, 2024 11:06:16.489    =
= [0x4f]    - KERNELBASE!CreateFileW    - 280340:802     - Thursday, November 21, 2024 11:06:16.536    =
= [0x50]    - KERNELBASE!CreateFileW    - 284118:6B      - Thursday, November 21, 2024 11:06:16.552    =
= [0x51]    - KERNELBASE!CreateFileW    - 28A0D7:6B      - Thursday, November 21, 2024 11:06:16.786    =
= [0x52]    - KERNELBASE!CreateFileW    - 28F4ED:6B      - Thursday, November 21, 2024 11:06:17.241    =
= [0x53]    - KERNELBASE!CreateFileW    - 293F4C:6B      - Thursday, November 21, 2024 11:06:17.416    =
= [0x54]    - KERNELBASE!CreateFileW    - 297832:26      - Thursday, November 21, 2024 11:06:17.495    =
= [0x55]    - KERNELBASE!CreateFileW    - 29957F:6B      - Thursday, November 21, 2024 11:06:17.558    =
= [0x56]    - KERNELBASE!CreateFileW    - 29C729:1DC     - Thursday, November 21, 2024 11:06:17.558    =
= [0x57]    - KERNELBASE!CreateFileW    - 29D0A2:13E     - Thursday, November 21, 2024 11:06:17.685    =
= [0x58]    - KERNELBASE!CreateFileW    - 29D287:1DC     - Thursday, November 21, 2024 11:06:17.685    =
= [0x59]    - KERNELBASE!CreateFileW    - 29DF92:1D6     - Thursday, November 21, 2024 11:06:17.558    =
= [0x5a]    - KERNELBASE!CreateFileW    - 29E02D:460     - Thursday, November 21, 2024 11:06:17.733    =
= [0x5b]    - KERNELBASE!CreateFileW    - 29F4C5:14A     - Thursday, November 21, 2024 11:06:17.780    =
= [0x5c]    - KERNELBASE!CreateFileW    - 29F573:138     - Thursday, November 21, 2024 11:06:17.780    =
= [0x5d]    - KERNELBASE!CreateFileW    - 2A009A:12F     - Thursday, November 21, 2024 11:06:17.812    =
= [0x5e]    - KERNELBASE!CreateFileW    - 2A05C1:138     - Thursday, November 21, 2024 11:06:17.828    =
= [0x5f]    - KERNELBASE!CreateFileW    - 2A0AAA:138     - Thursday, November 21, 2024 11:06:17.844    =
= [0x60]    - KERNELBASE!CreateFileW    - 2A0D07:138     - Thursday, November 21, 2024 11:06:17.844    =
= [0x61]    - KERNELBASE!CreateFileW    - 2A1206:27B     - Thursday, November 21, 2024 11:06:17.860    =
= [0x62]    - KERNELBASE!CreateFileW    - 2A1621:138     - Thursday, November 21, 2024 11:06:17.860    =
= [0x63]    - KERNELBASE!CreateFileW    - 2A1E0A:51      - Thursday, November 21, 2024 11:06:17.875    =
= [...]     -                           -                -                                             =
========================================================================================================
```

Let’s navigate to the first call to `CreateFileW` and check which filename was created on disk.

![image](https://github.com/user-attachments/assets/dbae0ea7-405c-4b88-a697-6ed0cddf4c94)


This will lead to this:

```
0:000> dx -s @$create("Debugger.Models.TTD.Position", 1824112, 1443).SeekTo()
(1008.1498): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 1BD570:5A3
```

Use the `k` command to display the call stack, where you’ll see `CreateFileW` at the top.

```
0:000> k
 # ChildEBP RetAddr      
00 00a3f5f4 0187fe8f     KERNELBASE!CreateFileW
WARNING: Frame IP not in any known module. Following frames may be wrong.
01 00a3f618 0188025c     0x187fe8f
02 00a3f694 0187fbbf     0x188025c
03 00a3f6e8 018804f8     0x187fbbf
04 00a3f708 01877a26     0x18804f8
05 00a3f740 01869b96     0x1877a26
06 00a3f788 0183dffd     0x1869b96
07 00a3f79c 0183df8f     0x183dffd
08 00a3f7b8 017e9778     0x183df8f
09 00a3f80c 017edd33     0x17e9778
0a 00a3f960 017ee12f     0x17edd33
0b 00a3fb08 017f3f60     0x17ee12f
0c 00a3fb40 01851e88     0x17f3f60
0d 00a3fb8c 00b5b941     0x1851e88
0e 00a3fbd8 00aa8934     Basta+0xcb941
0f 00a3fc24 00ab3818     Basta+0x18934
10 00a3fc70 75e77ba9     Basta+0x23818
11 00a3fc80 7748c0cb     KERNEL32!BaseThreadInitThunk+0x19
12 00a3fcd8 7748c04f     ntdll!__RtlUserThreadStart+0x2b
13 00a3fce8 00000000     ntdll!_RtlUserThreadStart+0x1b
```

Now that we’re at the point where `CreateFileW` is invoked, you can use the `!pde.dpx -du` command to display various types of information, such as the stack or memory around specific addresses. The `-du` is used to display Unicode strings. Here, you can see that a file named `fkdjsadasd.ico` was created in `C:\Users\WDAGUT~1\AppData\Local\Temp`

```
0:000> !pde.dpx -du
Start memory scan  : 0x00a3f5f8 ($csp)
End memory scan    : 0x00a40000 (User Stack Base)

0x00a3f5fc : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f620 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f6a4 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f6c0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f6f0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f714 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f748 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f774 : 0x01489202 :  !du ":\Users\WDAGUT~1\AppData\Local\Temp\"
0x00a3f790 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f7a4 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f7b0 : 0x01489202 :  !du ":\Users\WDAGUT~1\AppData\Local\Temp\"
0x00a3f7c0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f7d0 : 0x01489202 :  !du ":\Users\WDAGUT~1\AppData\Local\Temp\"
0x00a3f7d4 : 0x00a3f9f0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f7dc : 0x014888f0 :  !du "fkdjsadasd.ico"
0x00a3f7e8 : 0x014888f0 :  !du "fkdjsadasd.ico"
0x00a3f7f0 : 0x01489202 :  !du ":\Users\WDAGUT~1\AppData\Local\Temp\"
0x00a3f7f4 : 0x00a3f9f0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f814 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f8f8 : 0x00a3f9f0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f918 : 0x01489200 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\"
0x00a3f930 : 0x00660046 :  !du "Ffkdjsadasd.ico"
0x00a3f934 : 0x0064006b :  !du "kdjsadasd.ico"
0x00a3f938 : 0x0073006a :  !du "jsadasd.ico"
0x00a3f93c : 0x00640061 :  !du "adasd.ico"
0x00a3f940 : 0x00730061 :  !du "asd.ico"
0x00a3f944 : 0x002e0064 :  !du "d.ico"
0x00a3f968 : 0x00a3f9f0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3f9f0 : 0x01463468 :  !du "C:\Users\WDAGUT~1\AppData\Local\Temp\fkdjsadasd.ico"
0x00a3fb18 : 0x014358dc :  !du "C:\Users\WDAGUtilityAccount\Desktop\Basta.exe"
0x00a3fb1c : 0x018b1b88 : 0x01893070 :  !du ""         (((((                  H""
0x00a3fc10 : 0x00b1df98 :  !du "MiniPath"
```

Ransomware often creates a Mutex to ensure that only one instance of the executable runs, preventing issues like double encryption or similar conflicts.

```
0:000> dx @$cursession.TTD.Calls("kernelbase!CreateMutexW").First().TimeStart.SeekTo()
(1008.1498): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 1BD56D:106
@$cursession.TTD.Calls("kernelbase!CreateMutexW").First().TimeStart.SeekTo()
```

This output shows the current CPU register state and the instruction being executed. The debugger has paused at the `CreateMutex` function in `KERNELBASE.dll`. The `eax` register holds the memory address where the Mutex name is stored.

```
0:000> r
eax=00a3f92e ebx=75e91100 ecx=00a3000b edx=00a3f960 esi=00a3f92c edi=00000000
eip=75357380 esp=00a3f910 ebp=00a3f964 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
KERNELBASE!CreateMutexW:
75357380 8bff            mov     edi,edi
```

You can confirm this yourself by executing the following command:

```
0:000> du @eax
00a3f92e  "ofijweiuhuewhcsaxs.mutex"
```

How did we determine it was the `eax` register? Primarily with the help of the `!pde.dpx -du` command.

```
0:000> !pde.dpx -du 1
Start memory scan  : 0x00a3f910 ($csp)
End memory scan    : 0x00a40000 (User Stack Base)
Values to scan     : 1 (0x1)

       eax : 0x00a3f92e :  !du "ofijweiuhuewhcsaxs.mutex"
```

Without diving too deeply, as this example is focused on showing you how to navigate a TTD trace, we’ll wrap up by identifying the ransomware extension used by this specific Black Basta sample.

```
0:000> dx @$cursession.TTD.Calls("ADVAPI32!RegCreateKeyExWStub").First().TimeStart.SeekTo()
(1008.1498): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 1BD580:E0D
@$cursession.TTD.Calls("ADVAPI32!RegCreateKeyExWStub").First().TimeStart.SeekTo()
```

In this case, the ransomware is encrypting files with the `.xuy08dak6` extension.

```
0:000> !pde.dpx -du 1
Start memory scan  : 0x00a3f8b8 ($csp)
End memory scan    : 0x00a40000 (User Stack Base)
Values to scan     : 1 (0x1)

       eax : 0x014862c8 :  !du ".xuy08dak6\DefaultIcon"
```
