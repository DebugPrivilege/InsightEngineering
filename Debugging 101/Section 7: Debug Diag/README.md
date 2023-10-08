# What is Debug Diag?

Debug Diag is a utility provided by Microsoft for collecting and analyzing data to troubleshoot issues related to application crashes, hangs, memory leaks, and performance problems. 
This tool allows you to analyze multiple dump files at the same time which can be useful when you're dealing with issues or when you need to compare multiple instances of a problem to identify common patterns or anomalies.

Debug Diag can be downloaded here: https://www.microsoft.com/en-us/download/details.aspx?id=103453

# How to use Debug Diag?

Open Debug Diag and go to 'Add Data Files' to choose the crash dumps you'd like to analyze. Debug Diag also offers various analysis rules, allowing for quick, automated analysis of multiple crash dumps simultaneously. Here is an example:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f35151d8-e845-4a48-86f5-ec213f47fbe5)


Once the analysis is complete, it will generate an HTML report for us.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/296ec5e1-f0c9-4a57-83a1-a7720e0b0f32)


If we scroll down in the report, we can see additional details that is similar to the output of the **`!analyze -v`** command in WinDbg.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8c4367b3-7c8d-4022-a6b7-5e0861815817)


Here's a sample call stack from the crashing thread found in a crash dump:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cac8e577-adcf-430d-86d7-e0327984322b)

