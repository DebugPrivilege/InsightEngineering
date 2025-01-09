# Summary

To store a memory dump locally each time an app crashes, you can create a `REG_SZ` value named `CorporateWerServer` with an empty string for its value in the registry at `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Windows Error Reporting`.

![image](https://github.com/user-attachments/assets/2852ab4f-9b42-460f-8f0f-2b74b7cc9bdf)

From now on, whenever Windows Error Reporting detects that an application has crashed, it will store the crash data locally on the machine at `C:\ProgramData\Microsoft\Windows\WER\ReportArchive`.

![image](https://github.com/user-attachments/assets/84516861-9537-42c5-bd3b-6441282346ad)

