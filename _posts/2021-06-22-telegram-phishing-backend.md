---
layout: single
classes: wide
title:  "Using Telegram as Phishing backend"
date:   2021-06-22 15:00:00 +0200
category: Phishing
tags: [telegram, phishing, bots]
---

The amount of [phishing](https://en.wikipedia.org/wiki/Phishing) has been increasing over the course of the last few years.
The attackers use all kind of tricks and techniques for their phishing sites.
Often they don't do the actual hard work themselves but buy phishing kits from third parties.

## Discovery

Yesterday I stumbled across a well-made phishing site for the German banking group "Volksbank".

![Phishing site](/assets/img/2021-06-22-telegram-phishing-backend/phishing-site.png)

I did what I usually do: enter some fake data and check out what the site does.
In this case, there was a pretty interesting behavior.
Since I use uMatrix, I am able to check what resources a site loads and where the site sends http requests to.

![uMatrix](/assets/img/2021-06-22-telegram-phishing-backend/umatrix.png)

What stands out immediately is the XHR to `api.telegram.org`.
That domain hosts the [Telegram Bot API](https://core.telegram.org/bots/api).
If you ever worked with that API you know that you need a bot token and a user_id to send data via the API.

So let's check the embedded JavaScript.

![Header of the JavaScript](/assets/img/2021-06-22-telegram-phishing-backend/js-header.png)

At first we can see that this is very likely to be part of a phishing kit.
There's an author by the name of `ZeROツ` and even a copyright notice.
Quite humorously to develop a phishing site that intention it is to harm people but at the same time insist on your copyright.

![Content of the JavaScript](/assets/img/2021-06-22-telegram-phishing-backend/js-body.png)

Further down we can see that the token and the fraudster's Telegram user ID are right there in plain text.
That's great! Because we can use both to clone the data they have stolen.

## Extracting user data from Telegram

Now, Telegram is a cloud messenger.
That means, all the messages (except secret chats) are stored in the cloud.
And as long as no message is deleted, they are accessible by all participants of a chat.
That also means that we can utilize the [forwardMessage](https://core.telegram.org/bots/api#forwardmessage) API method to forward the conversation between the fraudster and the bot to our account.

For this purpose, I created a small python script.
You can find the code on [GitHub Gist](https://gist.github.com/d-Rickyy-b/405c2341d762fa87e73edd8f584830e6).
I updated the script several times since the release of this post.
Hence, the following code is outdated.
Make sure to check out the GitHub Gist linked above.

```python
# Script to forward the content of a chat between a Telegram Bot and a user to another user
# Make sure to insert your user_id into `receiver_id` and send /start to the bot on Telegram beforehand
import requests
import time

# from_chat_id and token are written in the javascript on the phishing site!
# the from_chat_id is the initial receiver of the phishing results
from_chat_id = ""
token = ""

# Enter your user_id as the receiver ID
receiver_id = ""

url = f"https://api.telegram.org/bot{token}/forwardMessage?chat_id={receiver_id}&from_chat_id={from_chat_id}&message_id="

error_counter = 0
message_counter = 0

while True:
    # Query Telegram to forward the message with the given ID to the receiver
    resp = requests.get(url + str(message_counter))

    if resp.status_code != 200:
        # If the Telegram API responds with a status code != 200, the message doesn't exist
        # That could mean that it was deleted, or it's the last message in that chat!
        # If there are 10 consecutive errors we stop the data collection!
        print(f"Issue with request at index {message_counter}! Stopping")
        error_counter += 1

        if error_counter >= 10:
            print("10 consecutive errors! Exiting!")
            exit()
    else:
        error_counter = 0

    # Telegram allows for 1 msg/s per chat - so we need this sleep in order to not get http 429 errors
    time.sleep(1)
    message_counter += 1

```

Now, after running that script, we copied the whole conversation to our own chat.
Btw, Telegram returns the message when calling the `forwardMessage` API endpoint, so you can also use the returned json object to further work with the exfiltrated data instead of going through it by hand.

![Stolen data](/assets/img/2021-06-22-telegram-phishing-backend/login-creds.png)

I handed the stolen data to the responsible banks and hope they'll contact the customers and double-check for weird transactions.
The customers need to change their credentials; otherwise, they are at risk of getting their accounts cleared.

## Conclusion

First of all, no, Telegram is not insecure because we are able to exfiltrate data.
We knew the bot token, which is like a password and should be kept secret by all means.
Saying Telegram is insecure because of this is exactly the same as saying, "Your bank account is insecure when I have your password".

### Why use Telegram?

For the attackers, it's easy and convenient to use Telegram.
You don't need to host anything yourself.
The backend can't be easily shut down, because it's hosted by Telegram.
Proxies are most likely allowing the connection, because it's not dangerous per se.
Also, the probability of being identified is close to zero. No domain or server is tied to your identity.
It's also extremely easy to deploy, because you only need a static site. This could make it even easier to deploy the phishing site on compromised hosts.

On the other hand, Telegram has the tools for us to extract their scraped data if they don't delete it.
This is different from the usual way of self-hosting your backend, where we usually don't get insight into the stolen data.

### Prevention

As you can see in an earlier screenshot, script blockers such as uMatrix are glorious tools for preventing such attacks.
uMatrix is especially good for creating rules for specific websites.
I can grant a website access to talk to the Telegram bot API (although I don't know any benign use case for that) while forbidding any connection for all other websites.
uMatrix makes it easy to give selected permissions to certain websites.

Also, of course, always make sure to check the URL of the site where you are about to enter your banking credentials. 
The URL of the described case was built like this: `kunden-volks.de.<random-domain-name>.cf/de/index.php?authId=804485`.
You can see that it tries to fool its visitors into believing they are on the domain `kunden-volks.de` with a URL like that `/<random-domain-name>.cf/de/index.php?authId=804485`. And I am sure that, for most users, it actually works.

### Files

You can check out the [main.js](/assets/files/2021-06-22-telegram-phishing-backend/main.js) file (used by the phishing site) yourself!
I replaced the original token and user_id with sample data.
