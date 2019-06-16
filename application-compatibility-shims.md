# Application Compatibility Shims

Application Compatibility Shims has been a popular persistence mechanism for at least a couple of years now and as our job is to emulate real world threats, I decided to spend some time learning how they worked, how they can be abused and how you can defend against them. Theres already a lot of great resources out there on this technique, but this article fills in some missing details that I encountered during my research.

### Application Shims <a id="application-shims"></a>

When Windows 95 came out there were a lot of changes to how Windows worked. To help make sure that programs continued to function properly, Microsoft created "Application Compatibility Shims". These shims allow users to essentially fake aspects of the environment for a given application, fixing issues that an application might encounter on a new version of Windows. For example, Shims offer the ability to do things such as tricking an application into thinking it has admin rights or redirecting file requests to a different folder.

In 2015 research started to come out on how these shims could be abused. An obvious example of this is by leveraging the "InjectDLL" option that shims provide. This functionality allows you to "backdoor" an application without actually editing the application itself, creating a stealthy method of obtaining persistence on a computer. With the shim in place, whenever the application is executed your payload will be executed as well.

[Casey Smith](https://twitter.com/subtee) recently posted a great article on using shims [to create in memory patches](http://subt0x10.blogspot.com/2017/05/using-application-compatibility-shims.html) and in it referenced an excellent walk through by [actae0n](https://twitter.com/actae0n) covering [how to create an Application Shim that uses InjectDLL](http://blacksunhackers.club/2016/08/post-exploitation-persistence-with-application-shims-intro/)

Actae0n's article is incredibly well written and helps to get you up and running with Application Compatibility Shims. I _highly_ recommend reading it. In working through it though, I ran into two issues:

1. My PoC DLL worked on the computer I developed it on, but would not work on a test PC.
2. I wanted to replace the MessageBox payload with a reverse shell

As someone with a tiny bit more than zero expierence with C++ and creating DLLs, the answers to these problems were not immediately obvious.

### Making the DLL work elsewhere <a id="making-the-dll-work-elsewhere"></a>

Testing with `rundll32.exe demo.dll <function>`, I knew I had a working MessageBox DLL, but if I tried to execute it on another computer in my lab I would get a 'specified module could not be loaded' error. After much digging, I found some posts on Stackoverflow suggesting that the cause of the issue was that the lab machine I was testing my payload on was [missing the VS C++ Redistributable package](https://stackoverflow.com/a/25685000). If you're developing legitimate software that's an easy fix but I really didn't want to have to install the C++ redistributable on every victim machine.

Further research let me to the `/MT` option in Visual Studio which will [bundle the necessary libraries into your binary.](https://stackoverflow.com/a/1610124). This bloats the size of the DLL a bit, but its a fair trade off in my opinion

### Making the PoC a payload <a id="making-the-poc-a-payload"></a>

Executing code seemed fairly start forward. I found the [ShellExecute](https://msdn.microsoft.com/en-us/library/windows/desktop/bb762153.aspx) function on MSDN and [some examples](https://stackoverflow.com/a/11566260). Problem is I couldn't get it to execute from DLLMain. Turns out [you're not supposed to](https://msdn.microsoft.com/en-us/library/windows/desktop/dn633971.aspx#general_best_practices).

DLLMain is designed to initialize an environment and not much else, but it _can_ create new threads. I decided to alter my approach and try [running ShellExecute from a new thread](https://stackoverflow.com/a/41730135) which worked!

### Final Payload <a id="final-payload"></a>

[This video](https://www.youtube.com/watch?v=afN0O9C-fBQ) demonstrates a Windows 10 computer on the left with a malicious shim for "PuTTY" \(a common program used by systems administrators\) installed. A fresh copy of putty is downloaded and then executed. Once putty executes the shim is triggered and a meterperter reverse shell is kicked off from the Windows computer to the Attackers computer on the right. This shell persists even after putty is closed.

### Indicators of Compromise <a id="indicators-of-compromise"></a>

In my research, I couldn't find a way to "disable" shimming, but it should at least be relatively easy to detect. There are two things that happen when a shim is installed

* `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Custom` and `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\InstalledSDB` are modified
* SDB files are placed in `%WINDIR%\AppPatch\Custom\`

Since Application Shims are rarely \(if ever\) installed in a production environment, either of the above events should be cause for an immediate investigation.

[Sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) can be used to monitor for changes to the above areas. Theres an old, unmerged pull request for SwiftOnSecurity's excellent [sysmon configuration](https://github.com/SwiftOnSecurity/sysmon-config/) that adds rules for Application Shims. I would download Tay's sysmon config from the master branch and and manually merge in the changes from [this pull request](https://github.com/SwiftOnSecurity/sysmon-config/pull/25/commits/fbcd7b540bd304929513ce26f3d5154fe3392b41)

