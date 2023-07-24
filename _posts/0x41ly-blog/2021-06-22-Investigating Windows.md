---
title: Investigating Windows
author: 0x41ly
classes: wide
ribbon: red
date: 2021-06-22 18:00:00 +0800
header:
  teaser: 'https://i.imgur.com/o37B2mV.png'
description: TryHackMe room.
categories: TryHackMe
toc: true
published: true
---


## **Hello Buddies ...**
​
### Welcome to the  first windows room in TryHackMe [check it](https://tryhackme.com/room/investigatingwindows)
​
## INTRO
### What is Windows server?
- Windows Server refers to any type of server instance that is installed, operated and managed by any of the Windows Server family of operating systems.
Windows Server exhibits and provides the same capability, features and operating mechanism of a standard server operating system and is based on the Windows NT architecture......[Read more.](https://www.techopedia.com/definition/359/windows-server)<br/>
<hr/>
## SUMMARY
- We gonna learn how to investigate windows.<br/>
<hr/>
Let's start...<br/>
It's important to read every task content..<br/>
<br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/1.png" width="600" > <br/>
<br/>
### Whats the version and year of the windows machine? <br/>
go to `start menu >> settings >> system >> about` <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/2.png"  > <br/>
Ans: Windows Server 2016 <br/>
<br/>
### Which user logged in last? <br/>
Actually it's the user we already logged with <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/3.png"  > <br/>
Ans: Administrator <br/>
<br/>
### When did John log onto the system last? <br/>
 open your cmd and type `net user John`<br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/4.png"  > <br/>
Ans: 03/02/2019 5:48:32 PM <br/>
<br/>
### What IP does the system connect to when it first starts? <br/>
well, I didn't know how to get that but nothing called i don't know so let's find out with our dear google... after some search about how to know the state of the machine when it's starts and it's the SOFTWARE  subkey of the HKLM and after more search [I found this document](https://docs.microsoft.com/en-us/windows/win32/setupapi/run-and-runonce-registry-keys) so run `cmd >regedit>Yes` then go to `Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/5.png"  > <br/>
Ans: 10.34.2.3 <br/>
<br/>
### What two accounts had administrative privileges (other than the Administrator user)? <br/>
`Right click on menu bar > Computer Managment > Local Users and Groups > Groups > Administartors ` <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/6.png"  > <br/>
Ans: Jenny, Guest <br/>
<br/>
### Whats the name of the scheduled task that is malicous.? <br/>
Just search for Task Scheduler in Task Scheduler Library a suspicious task running nc.ps1 it's a net cat <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/7.png"  > <br/>
Ans:  clean file system <br/>
<br/>
### What file was the task trying to run daily? <br/>
We knew from previous question its nc.ps1 <br/>
Ans:  nc.ps1 <br/>
<br/>
### What port did this file listen locally for? <br/>
Ans:  1348 <br/>
<br/>
### When did Jenny last logon? <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/8.png"  > <br/>
Ans:  Never <br/>
<br/>
### At what time did Windows first assign special privileges to a new logon? <br/>
[I found an interesting site ](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/default.aspx) I searched in the site for assign special privileges to a new logon<br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/11.png"  > <br/>
so its event id 4672. On windows Search for  `event viewer > secuity ` search for the event id 4672. <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/10.png"  > <br/>
Ans:  03/02/2019 4:04:49 PM <br/>
<br/>
### At what date did the compromise take place? <br/>
Ans:  03/02/2019  <br/>
<br/>
### What tool was used to get Windows passwords?<br/>
In Task Scheduler there is another suspicious task called GameOver <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/12.png" width="600"  > <br/>
Ans:  Mimikatz <br/>
<br/>
### What was the attackers external control and command servers IP?<br/>
Let's check hosts file go to  `C:\Windows\System32\drivers\etc` open hosts ile with notepad. <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/13.png"  > <br/>
Ans:  76.32.97.132 <br/>
<br/>
### What was the extension name of the shell uploaded via the servers website?<br/>
Since default webserver is IIS so I went to `C:\inetpub\wwwroot\`   <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/14.png"  > <br/>
Ans:  .jsp <br/>
<br/>
### What was the last port the attacker opened? <br/>
Go to Windows Defender Firewall with Advanced Security the inbound rules and check the port of first rule   <br/>
<img src="{{site.baseurl}}/assets/images/0x41ly-blog-img/windows1/15.png" width= "1000"  > <br/>
Ans: 1337 <br/>
<br/>
### Check for DNS poisoning, what site was targeted? <br/>
It was in hosts file   <br/>
Ans:  google.com <br/>
<br/>

