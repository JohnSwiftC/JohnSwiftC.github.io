---
layout: post
title:  "UAC Bypassing With fodhelper.exe, even after patches."
date:   2024-12-28 12:00:00 -0600
categories: cybersecurity maldev networking
---

# First, what is the OG bypass?

Back in the olden days, people loved to walk right past UAC with Registry Key Manipulation and fodhelper.exe. In short, fodhelper.exe will run processes with the highest elevation that are designated in the reg key `HKCU:\Software\Classes\ms-settings\Shell\Open\command`, in the `(default)` value, if the `DelegateExecute` value is set. This is obviously pretty bad for UAC and since then Microsoft has been hard at work (not really) trying to solve this problem.

After various security updates, times are different now. If you attempt to do the same thing now with a simple PowerShell script, Defender blows up, closes the calling process, and even resets our registry keys to default, quite a shame.

Here is a sample script that does this. (Remember this, it's important later 😉)

{% highlight powershell %}
[String]$maliciousExePath = "C:\whatever"

New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value $maliciousExePath -Force

Start-Process "C:\Windows\System32\fodhelper.exe"
{% endhighlight %}

# Initial Observations

Playing around with this technique, even in its broken state, I found some fun stuff. Maybe you as the reader can figure out where I'm going with this:

1. The process modifying registry keys is not *instantly* closed, so the PowerShell script above will still (most of the time) execute the line under it that starts fodhelper.exe.
2. The process started maliciously is closed, but also not *instantly*.
3. The new registry value is not *instantly* reset. This should be kind of obvious, seeing that the malicious process is started at all by the above script. This face proves especially useful later, however.

# Discovery

I have been developing my own reverse shell (slowly becoming a full RAT), GuShell, for a minute now, so I decided to test that. When I utilised the listener, I noticed that I achieved the connection for a small period of time before the process was closed. This let me know that GuShell was being ran, even if it's just for a little bit.

Bear with me here now, GuShell has functions that both move itself, add itself to local machine autostart reg keys, and then also use the `DisableAntiSpyware` value to disable Windows Defender. These steps typically require elevation to do, and we get that elevation in the little time I have to do anything.

I then used that little time to call both of these functions right at main. To my absolute delight, it worked! Defender could not close the process in time and all the registry keys were added. Here is where the magic comes in. Remember how the value that messes with fodhelper.exe is reset? After adding the `DisableAntiSpyware` value, Defender *completely fails* to reset it. So, we now have Defender disabled, autostart enabled, a hidden piece of malware, and a direct way to elevate it. Fun stuff!

# Technique Simplified (How I am using it)

I also discovered that if you create a PowerShell instance with `CreateProcess`, that only that new child will be killed when using the malicious script. Knowing this...

In my malware:

1. Create a child PowerShell process through whatever means (probably create a script in directory to execute.)
2. Start evil script that opens the malware *again*, this time with the few seconds of elevated privilege.
3. Malware always attempts to `DisableAntiSpyware` when ran, and does so with the small amount of time allowed.
4. We now are able to open the fodhelper.exe process, which will start the malware with complete elevation.

# Protecting yourself from this bypass

It is kind of easy, just set the UAC settings to ALWAYS ASK! For some reason, Windows does not normally do this and it definitely should. This setting will stop this bypass in its tracks!

# Disclaimer

Bypassing UAC on machines that you do not have permission to do so on is illegal. Think about the possible ramifications and effects that you may have if you use this on a vulnerable machine.

As always, I write these for security education and fun, and do not intend for any harm to be dome.