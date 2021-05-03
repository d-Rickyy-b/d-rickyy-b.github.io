---
layout: single
classes: wide
title:  "Telegram account hijacking via bots"
date:   2021-05-02 20:30:00 +0200
category: Malware
tags: [telegram, scam, bots]
---

Today a user contacted us in [@BotTalk](https://t.me/BotTalk) on Telegram. He wrote that his bot was sending "weird messages". I had a short chat with him about it and it quickly turned out that he's not the only one with the issue.

![Quick search within the BotTalk group](/assets/img/2021-05-02-telegram-account-hijacking/bottalk_search.png)

So I decided to look into it.

# Symptoms
His bot started sending weird messages such as these:
```
Stand By ...
```

```
Hi There!

Before We Can Continue We Need To Verify That You're A REAL User
```

```
Please Type the code you've just recieved via SMS In The OnScreen Keyboard Below
```

In our case the previously mentioned texts were sent from the bot's account, even when the actual code of the owner of the bot wasn't running.

That means someone was running code with the same bot token! At first we thought it was a Telegram feature. But upon further inspection we found that it very likely was not!

![Bot conversation](/assets/img/2021-05-02-telegram-account-hijacking/bot_conversation.png)


# Explanation
Let's check out what exactly this is and how this whole thing is supposed to work.

## Scam workflow
Let me shortly explain how this fraud works:

1. Victim (V) contacts bot
2. Adversary (A) replies to user and asks the user to "verify"
3. (A) requests phone number
4. (V) sends phone number
5. (A) sends login request to Telegram
6. Telegram sends login code to (V)
7. (V) types login code via inline keyboard
8. (A) logs into the Telegram account of the victim

Potentially there is an additional step for asking for the 2FA password - I wasn't able to test that.
In our case the login came from an IP address located in Romania. Seemingly from an Android device.

### Graphical Overview
Below you can find a detailed graph on all the entities involved.

<figure>
  <a href="/assets/img/2021-05-02-telegram-account-hijacking/scam_workflow_full.png">
    <img src="/assets/img/2021-05-02-telegram-account-hijacking/scam_workflow_full.png" alt="Overview of the scam workflow">
  </a>
</figure>

### Workflow visualized as a timeline

<figure>
  <a href="/assets/img/2021-05-02-telegram-account-hijacking/scam_workflow_time.png">
    <img src="/assets/img/2021-05-02-telegram-account-hijacking/scam_workflow_time.png" alt="Timeline of the scam workflow">
  </a>
</figure>

## ⚠️ Consequences
After logging in, the adversary has full access to your Telegram account, can access all your Telegram "Cloud Chats" (which are all chats except the "Secret Chats") and the messages you sent in the past. They can send new messages to your contacts and beyond. They can delete conversations. They can make you join groups, channels, etc. They can set up 2FA and lock you out of your account.

Also the adversaries could send illegal content (e.g. stuff forbidden by the [Telegram ToS](https://telegram.org/tos)). The Telegram abuse team will ban your/the bot owner's phone number from using Telegram. Forever.

## How was the bot compromised?

There are at least three ways on how a Telegram bot could get compromised. Both are very trivial.

1. The bot is owned by the adversary
2. The **bot's token** of a random other bot was published to e.g. GitHub, Pastebin or any other place
3. The bot owner of a random other bot fell for the previously explained scam themselves and gave access to their account

With access to the account (2) the adversaries could simply read the chat with [@BotFather](https://t.me/BotFather) and get hold of the bot token.

## How to take back control...

### ... when your account was hijacked?
1. Open your Telegram Settings > Privacy & Security > Sessions
2. Kill all the sessions you don't know!
3. Set up a 2FA password so that the adversary can't log in again

### ... when your bot was hijacked?
1. Revoke the token via [@BotFather](https://t.me/BotFather)
2. Make sure you don't commit your token again. E.g. put it into a separate file and add it to your `.gitignore`!

# Conclusion & How to prevent
Summing up: If your bot is replying to messages with either of the initially mentioned messages (and you did not add these yourself) then someone had access to your bot token!

**Never ever make your bot token public**. Treat it as if it was a password to a valuable account. Also N**EVER give out your Telegram login codes**. 

Also absolutely make sure to **enable the Telegram Two-Factor Authentication (2FA)** for your account. That way the adversaries don't only need your login code but also a password. This makes it harder for them to access your account.
