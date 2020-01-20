---
layout: single
classes: wide
title:  "RoboTheifClient - A Telegram session stealer"
date:   2020-01-20 21:00:00 +0200
category: Malware Analysis
tags: [malware, telegram, session stealer, .Net]
---

While scrolling through a Telegram group, I recently (late december 2019) found a tool that claimed to add fake followers to your Telegram channels. That was a pretty provoking claim, so I downloaded the tool and analyzed it. Here's what I found.


# Analysis of the file
At first we can check the file type and we immediately notice that it's a .Net binary. That's awesome, because you can get pretty much decompile the source code of a binary with tools such as `.NET Reflector` and hence can easily check what that binary is doing.

```bash
$ file apiadder.exe
apiadder.exe: PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows
```

So next up we load that binary into `.NET Reflector` and see what we got there. At first it checks if any arguments are passed to the binary. If there are no arguments, the binary will create a new process from the binary itself with "MM" as the arguments and `Hidden` WindowStyle, that hides the processes window, making it invisible for a normal user. After that happened, the initial program kills itself. The new process obviously got one argument passed, so it executes the `if` branch. And that's where things start to become interesting.

```
private static void Main(string[] args)
{
    if (args.Count<string>() != 0)
    {
        TelegramBot.SetValues();
        InitBot();
        FindTelegram();
        Console.ReadKey();
    }
    else
    {
        Process process = new Process();
        ProcessStartInfo info = new ProcessStartInfo {
            Arguments = "MM",
            WindowStyle = ProcessWindowStyle.Hidden,
            FileName = Assembly.GetExecutingAssembly().Location,
            CreateNoWindow = true,
            UseShellExecute = true
        };
        process.StartInfo = info;
        process.Start();
        Process.GetCurrentProcess().Kill();
    }
}
```

We quickly come to the conclusion that it's indeed a Telegram related tool, because it initializes a TelegramBot object and tries to find a running Telegram Desktop process. 

![Finding the tdata directory](/assets/img/2020-01-20-telegram-session-stealer/tdata-directory.png)

After it did that, it searches for the Telegram directory (where the binary and the session data is stored in) and after it found that directory, it starts compressing the `tdata` directory. But for that to work it needs to kill the Telegram process, because Telegram is still holding file locks to files in the `tdata` directory. So it kills Telegram and compresses the folder. It stores the compressed result to the system's Temp directory in a file in the called `MT.zip`. That zip file will be encrypted by a very weak password: `MM` (yes, literally those two letters).

![Object browser of .Net Reflector](/assets/img/2020-01-20-telegram-session-stealer/reflector-object-browser.png)

After the zip file was created, it starts Telegram again so that the user won't become too sceptical of what is happening on his computer. And right after that it sends the zip file via the Telegram API to a user, that has been 

**Fun fact:** the `Console.ReadKey();` you saw in the `Main` method is being used to prevent the program from ending itself before the upload has finished. The upload is done in an async way (another thread) and thus the program would reach the end of Main and hence just close itself. The `Console.ReadKey();` method blocks the current thread waiting for any input (to the hidden window). That gives more than enough time for the upload to finish.


![Clear text bot token and userID](/assets/img/2020-01-20-telegram-session-stealer/bottoken-and-id.png)


Luckily both bot token and userID are stored in the binary itself, unobfuscated. I removed the bot token from this blog post nevertheless. We can now utilize the [Telegram bot API](https://core.telegram.org/bots/api) ourselves and receive information about the bot.

```bash
$ curl https://api.telegram.org/bot<token>/getMe
{"ok":true,"result":{"id":650392548,"is_bot":true,"first_name":"Members Adder","username":"OfficialMembersBot"}}
```
We can also investigate more information about the receiver of all those stolen Telegram sessions - we got his userID.

```bash
$ curl https://api.telegram.org/bot<token>/getChat?chat_id=535163757
{"ok":true,"result":{"id":535163757,"first_name":"\u0299\u029c\u1d1c\u1d0b\u1d0b\u1d00\u1d05 s\u1d00\u029f\u1d00","username":"<removed>","type":"private","photo":{"small_file_id":"...", [...]}}}
```

Well, obviously we can now report the bot and the user to [@NoToScam](https://t.me/notoscam) on Telegram or use the in-app report feature to report the bot. That way they *might* be stopped from scamming users.

It seems that by now the session stealer was already open-sourced by the creator. At least there is a GitHub repository, where you can find the malware: [RoboThief-Telegram-Session-Stealer](https://github.com/MrModed/RoboThief-Telegram-Session-Stealer).

# YARA rule for RoboThiefClient
After that analysis I wanted to be able to find other occurrances of that malware in a binary, so I wrote my first real [YARA](https://github.com/VirusTotal/yara) rule. I didn't have any (real-world) experiences with YARA yet. If you don't know YARA yet, it's best to read the description text on the [YARA project website](https://virustotal.github.io/yara/):

*YARA is a tool aimed at (but not limited to) helping malware researchers to identify and classify malware samples. With YARA you can create descriptions of malware families (or whatever you want to describe) based on textual or binary patterns. Each description, a.k.a rule, consists of a set of strings and a boolean expression which determine its logic.*


```yara
rule RoboThiefClient {
	meta:
		author = "Rico - @0Rickyy0"
		reference = "https://www.virustotal.com/gui/file/2ad784eeb5525723162c521b9aefac965578d77d6269619abfb8656c25e9c48b/detection"
		reference = "https://www.virustotal.com/gui/file/916db604bd5a3132cffc4c7a266585579fc9a5838ec0eba11f3532de5902c901/detection"
		reference = "https://t.me/C0D3BA5E/99"
		reference = "https://github.com/MrModed/RoboThief-Telegram-Session-Stealer"
		date = "2020-01-20"
		description = "This rule detects the telegram credential stealer 'RoboThiefClient'"
		
	strings:
		$robothiefclient = "RoboThiefClient" ascii wide nocase
		$tdata = "tdata" ascii wide
		$zip_file = "MT.zip" ascii wide 
		$bottoken = /[0-9]+:[a-zA-Z0-9\-_]+/ ascii wide
		
	condition:
		uint16(0) == 0x5a4d and filesize < 1000KB and $robothiefclient and 2 of ($zip_file, $tdata, $bottoken)
}
```

That rule uses some patterns such as the hardcoded name of the zip file `MT.zip` and names such as `tdata` or even the malware's own name "RoboThiefClient". But only if the file starts with the typical MZ header and has a filesize of below 1 MB. I also implemented a regex scan for a (API) bot token which can use up a lot of ressources when scanning large files. Since we limited the filesize this should be okay to use.


# Conclusion & How to prevent
As always please be aware that every piece of software you run on your computer can potentially be malware. Don't click on random stuff you get sent on the internet, even if it's a friend sending it to you. And never believe stuff that promises you to give you stuff for free. Or which promises you to get instantly more followers.

We still don't know what these guys are doing with those stolen sessions then. I would bet that those are used for spreading spam in the name of another user. But there is a way to protect bad guys from stealing your session: Make sure to enable "Local passcode" in the Telegram settings > Privacy and Security > Local passcode. That setting uses your passcode to encrypt the session data in tdata. That way the current implementation of the malware just sends the encrypted data to the adversary. Without the passcode they are pretty lost.

![Enable local passcode in Telegram](/assets/img/2020-01-20-telegram-session-stealer/telegram-passcode.png)

I hope you enjoyed this short, quick-and-dirty analysis.


# References:
- [Virustotal Sample 1](https://www.virustotal.com/gui/file/916db604bd5a3132cffc4c7a266585579fc9a5838ec0eba11f3532de5902c901/detection)
- [Virustotal Sample 2](https://www.virustotal.com/gui/file/2ad784eeb5525723162c521b9aefac965578d77d6269619abfb8656c25e9c48b/detection)
- [RoboThiefClient - GitHub Repo](https://github.com/MrModed/RoboThief-Telegram-Session-Stealer)
- ⚠️ [Archive of the malicious files I analyzed](/assets/files/2020-01-20-telegram-session-stealer/RoboThiefClient.zip) - password is `infected`
