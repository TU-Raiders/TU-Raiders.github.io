---
title: Process Handler
author: 0x41ly
classes: wide
ribbon: red
date: 2021-07-16 18:00:00 +0800
description: How to make a malware series.
categories:
  - Exploit Development
toc: true
published: true
---

## Hi guys
Welcome back :)  

Our task today is to retrieve HANDLE for any process.
## What is HANDLE?
![](/assets/images/0x41ly-blog-img/handle/0.png)
A process handle is an integer value that identifies a process to Windows. The Win32 API calls them a HANDLE; handles to windows are called HWND and handles to modules HMODULE.<br>
Threads inside processes have a thread handle, and files and other resources (such as registry keys) have handles also.
The handle count you see in Task Manager is "the number of object handles in the process's object table". In effect, this is the sum of all handles that this process has open. I strongly recommend reading [Process Handles and Identifiers](https://docs.microsoft.com/en-us/windows/win32/procthread/process-handles-and-identifiers). <br>

First we have [GetCurrentProcessId()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getcurrentprocessid) function which retrieves the process identifier of the calling process.

```
#include <windows.h>
#include <iostream>
using namespace std;

int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)

{
 DWORD Pid = GetCurrentProcessId();
 cout<< "The PID of current process is " << Pid<<endl;
 system("pause");
 return 0;
}
```
![](/assets/images/0x41ly-blog-img/handle/1.png)

Now let's try [GetCurrentProcess()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getcurrentprocess)
```
#include <windows.h>
#include <iostream>
using namespace std;

int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)

{
 DWORD Pid = GetCurrentProcessId();
 cout<< "The PID of current process is " << Pid<<endl;
 HANDLE h=GetCurrentProcess();
 cout<<"HANDLE IS: "<<h<<endl;
 system("pause");
 
 return 0;
}
```
![](/assets/images/0x41ly-blog-img/handle/2.png)
It returned 0xffffffffffffffff which is -1. I went back to the function documentation found that.
![](/assets/images/0x41ly-blog-img/handle/3.png)

Let's try [DuplicateHandle()](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-duplicatehandle).

```
#include <windows.h>
#include <iostream>
using namespace std;

int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)

{
DWORD Pid = GetCurrentProcessId();
 cout<< "The PID of current process is " << Pid<<endl;
 HANDLE hRealHandle = 0;
 DuplicateHandle( GetCurrentProcess(), // Source Process Handle.
                 GetCurrentProcess(),  // Source Handle to dup.
                 GetCurrentProcess(), // Target Process Handle.
                 &hRealHandle,        // Target Handle pointer.
                 0,                   // Options flag.
                 TRUE,                // Inheritable flag
                 DUPLICATE_SAME_ACCESS );// Options
 cout<<"Real handle is: "<<hRealHandle;
 return 0;
}
```
![](/assets/images/0x41ly-blog-img/handle/4.png)

So all that was for the current process, then how to do the same for others? <br>

First, let's find a way to get PID of any process, I found a way to do that with the process name. let's try it.
```
#include <windows.h>
#include <iostream>
using namespace std;
#include <tlhelp32.h>




int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)
{
	PROCESSENTRY32 entry;
	entry.dwFlags = sizeof(PROCESSENTRY32);

	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS,NULL);

	if(Process32First(snapshot,&entry) == TRUE){
		while(Process32Next(snapshot,&entry) == TRUE){
			if(_stricmp(entry.szExeFile,"cmd.exe") == 0){          
				cout << "PID: " << entry.th32ProcessID;	
			}
		}
	}
	CloseHandle(snapshot);
	return 0;
}

```
 We are searching for PID of cmd.exe, let's compile and run.
 ![](/assets/images/0x41ly-blog-img/handle/5.png)

So for flexibility, we gonna turn it into function.
```

#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
using namespace std;



DWORD GetPid(const char* ProcessName)
{
	PROCESSENTRY32 entry;
	entry.dwFlags = sizeof(PROCESSENTRY32);

	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS,NULL);
	DWORD x;

	if(Process32First(snapshot,&entry) == TRUE){
		while(Process32Next(snapshot,&entry) == TRUE){
			if(_stricmp(entry.szExeFile,ProcessName) == 0){ 
				x= entry.th32ProcessID;
			}
		}
	}
	CloseHandle(snapshot);
	return x;
}


int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)
{
	DWORD PID = GetPid("cmd.exe");
	cout<< "PID: " <<PID<<endl;
}
```
 ![](/assets/images/0x41ly-blog-img/handle/6.png)
 
 Actually, in that way, we are just going to have the last PID of the desired process. For example, it is not a must that "cmd.exe " only running with one PID it may have many PIDs, so we need to change our code a little bit to get all PIDS. 
 ```
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
using namespace std;



DWORD * GetPid(const char* ProcessName)
{
	PROCESSENTRY32 entry;
	entry.dwFlags = sizeof(PROCESSENTRY32);

	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS,NULL);
	static DWORD x[200];
	int i=1;
	if(Process32First(snapshot,&entry) == TRUE){
		while(Process32Next(snapshot,&entry) == TRUE){
			if(_stricmp(entry.szExeFile,ProcessName) == 0){  
			x[i]= entry.th32ProcessID;
			i +=1;        
				
			}
			x[0]=i;
		}
	}
	CloseHandle(snapshot);
	return x;
}


int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)
{
	DWORD *PID_arr;
	PID_arr = GetPid("cmd.exe");
	DWORD size = *PID_arr;

	DWORD PID;
	for (DWORD i = 1; i < size; ++i) {
    	PID = *(PID_arr + i);
	cout<< "PID: " <<PID<<endl;
 
   }
	

   return 0;             
}
 ```
  ![](/assets/images/0x41ly-blog-img/handle/8.png)
 
Now we gonna use [OpenProcess()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) to retrieve HANDLE for cmd.exe
 
 ```
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
using namespace std;



DWORD * GetPid(const char* ProcessName)
{
	PROCESSENTRY32 entry;
	entry.dwFlags = sizeof(PROCESSENTRY32);

	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS,NULL);
	static DWORD x[200];
	int i=1;
	if(Process32First(snapshot,&entry) == TRUE){
		while(Process32Next(snapshot,&entry) == TRUE){
			if(_stricmp(entry.szExeFile,ProcessName) == 0){  
			x[i]= entry.th32ProcessID;
			i +=1;        
				
			}
			x[0]=i;
		}
	}
	CloseHandle(snapshot);
	return x;
}


int WINAPI WinMain (HINSTANCE hThisInstance, HINSTANCE PrevInstance,
                            LPSTR lpszArgument, int nFunsterStil)
{
	DWORD *PID_arr;
	PID_arr = GetPid("cmd.exe");
	DWORD size = *PID_arr;

	DWORD PID;
	for (DWORD i = 1; i < size; ++i) {
    PID = *(PID_arr + i);
	cout<< "PID: " <<PID<<endl;
	HANDLE hProcess = OpenProcess(PROCESS_QUERY_INFORMATION,FALSE,PID);
	cout<<"HANDLE: "<<hProcess<<endl;
	CloseHandle(hProcess);  
   }
	

   return 0;             
}

 ```
 
  ![](/assets/images/0x41ly-blog-img/handle/7.png)
  
  
  
  We did it ;).

  
  

