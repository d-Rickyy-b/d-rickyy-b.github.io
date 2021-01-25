---
layout: single
classes: wide
title:  "Quote unquote - How not to encode passwords"
date:   2021-01-22 10:00:00 +0200
category: Debugging
tags: [Android, Debugging]
---
# My approach to debugging an issue in Outlook

## tl;dr
I found a bug (to be confirmed) in the Outlook app for Android.
The app is unable to handle double quotes in passwords correctly.
Talking to support is a mess.


## Introduction
Some times ago I searched for a new email client for my Android device.
After a bit of a search, I found Outlook for Android and basically stuck with it for the last 2 or so years.
It's a solid app, modern looks, some nice features. It's decent.
So what would you want to do with an email client? Of course: Add your mail accounts to it so that you can read all your mails.
That worked for most of my accounts - as expected.
Only for a single account Outlook would repeatedly tell me that it's the wrong mail/password combination.
I am using a password manager, so it would be really odd if it actually was the wrong password (which btw. works fine in other apps).
Being a curious person, I couldn't just accept the issue or simply report it, so I decided to debug this issue.
In this post I want to try to show you how I approached finding the mentioned issue step-by-step.


## Localizing the issue
When you report a bug, the most important part is to describe it precisely as possible.
The more specific the report is, the better.
So the first thing is to find out when exactly the bug occurs.  
It was clear that the issue was somewhat related to the account in question: `spam@rico-j.de`.
Another account on the same domain worked fine for months, so I was pretty sure this wasn't a server configuration issue.
Thus a logical step would be to compare the client configurations for both accounts.
And I did that, but there really was no difference.


### Wrong password? Typo? Copied another character?
Alright, what else could be the reason?
Maybe the password really is wrong?!
So I double checked my password manager.
I copied the password into the fully visible "account description" field so that I could compare it before actually logging in.
Well, the password was correct - but it still failed to log in.


### Bug in the app?
Sooo... what if the Outlook app actually had a bug during parsing and never sent the password? Or perhaps it did send the password but kind of "in a wrong way"? What if the app sends a wrong password?
One should think that an app with 100.000.000+ downloads on the Google Play Store shouldn't have issues with certain special (ascii!) characters, right?

**Wrong!**
At the time that I realized there must be an issue with the app itself, I contacted the in-app support and tried to explain the situation to them.
Of course they asked me to do all kinds of things (switching from Wi-Fi to mobile, clearing the app's cache, etc.). The only thing missing was "have you tried turning it off and on again?" 😉.

I don't want to blame them, they don't know in detail what you already tried on your end and probably need to help out several hundreds of (non-technical) users each day.
However it's so energy-consuming to try to get your issue across and the other person strictly follows a script although you clearly showed that you are 5 steps ahead of their script.

Well, time to find some more details on the issue so that I can tell the support team to escape the L1 hell.


### Wrong password? Well yes, but actually no (but actually yes)
As written before, there was another account that worked fine and it only differed in a single aspect: the password.
I compared the range of characters that has been used for both accounts.
The `spam@rico-j.de` one used several extra chars: `|+="`.

With my knowledge about general parsing issues of all kinds, I decided to simply remove all those "special" characters from my password altogether.

![I'm in](/assets/img/2021-01-22-outlook-for-android-bug/im_in.jpg)

It worked 🎉! So technically the password was "wrong". 
The problem is just very unlikely to be on my end.
So which was the character that caused all of this?
I changed the password several times, using only one of the above characters at a time.
The login worked for each of them, except for the double quote character: `"`.

Of course I immediately sent this information to the support team.
Their reply was "please file a feature request if you think it'd be a nice addition".
Uhm, not sure about you, but I'd love being able to use more than just a-z, A-Z and 0-9 in my passwords.
I do not consider this a missing feature. It's an essential **bug**.
I started the chat again, and after a bit of convincing how this is actually a bug, they started working on it and finally passed it to L2.


## Bug Hunt
Reporting such an issue with vague knowledge about it didn't suffice me.
I wanted to know the exact reason for this to happen.
Can't be too hard to debug this, right? Spoiler: It honestly wasn't that hard.
With the input of some smart people, I came up with the idea to set up a honeypot.
Usually honeypots are being used to monitor attacks against certain (mostly internet connected) hard- or software. 

But in this case, however, I just wanted to observe what the plain text password that the server receives looks like.
I already host my own mail server, so I didn't need to do any server or DNS configuration whatsoever.
A quick Google search threw ["heralding"](https://github.com/johnnykv/heralding) right at me.
It's a python project that runs honeypots on all kinds of ports and for all kinds of services.
So I shut down my mail server and fired up "heralding" which immediately starts listening on multiple ports, such as 143 and 993  (IMAP).
With that we're able to sniff on the raw IMAP commands the server is receiving from the mailing app(s).

To have another result as comparison, I downloaded the K9 mail app and tried to log in to the server with both apps.
As password I used `aaaa"bbbb` both times.

**K9 Mail:**
```
3 LOGIN "spam@rico-j.de" "aaaa\"bbbb"
b'3 LOGIN "spam@rico-j.de" "aaaa\\"bbbb"\r\n'

```

**Outlook for Android:**
```
A4054 LOGIN spam@rico-j.de "aaaa"bbbb"
b'A4054 LOGIN spam@rico-j.de "aaaa"bbbb"\r\n'
```

The first line of each of the prior code blocks is the received **IMAP command** as a string. The second one is the same command as a python binary string.

We can instantly see that outlook lacks a backslash character `\` right in front of the double quote `"`.
This is it. 
That's the reason why I couldn't login with the `spam@rico-j.de` account.
And btw. using "K9" to log in to the actual mail server, the login unsurprisingly succeeds without any problems.


### Oh my RFC
But what format exactly would a mail server expect?

As with (nearly) all things on the internet, there's an RFC describing the underlying technology.
In the case of mailing, the protocol we are looking for is IMAP.
The proper RFC is [RFC3501](https://tools.ietf.org/html/rfc3501) and in [section 9](https://tools.ietf.org/html/rfc3501#section-9) it is specifically talking about the syntax described in the ABNF form.
Here we can find the format of a password.


```
password        = astring

astring         = 1*ASTRING-CHAR / string

string          = quoted / literal

literal         = "{" number "}" CRLF *CHAR8
                    ; Number represents the number of CHAR8s

quoted          = DQUOTE *QUOTED-CHAR DQUOTE

QUOTED-CHAR     = <any TEXT-CHAR except quoted-specials> /
                  "\" quoted-specials

quoted-specials = DQUOTE / "\"
```

Strings can be defined on two ways via IMAP - quoted or literal.
The representations we saw earlier were both "quoted".

Referring to the ABNF specification above, our `password` is basically a `string`. 
That again can be either a `literal` or a `quoted` string.
We are using `quoted`, so our password should have the general format of `"password"` - **but** `QUOTED-CHAR` clearly states that `quoted-specials` (the chars `"` and `\`) must be prepended with a backslash `\`.

So the correct syntax for the password would be `aaaa\"bbbb` - just the way k9 is doing it.

## Conclusion
Outlook doesn't adhere to the specs defined in RFC3501 when it comes to encoding passwords (and probably even other strings).
I need to admit that my knowledge of IMAP is very limited, so I don't know all the possible effects this could have.
One effect is clearly that quoted passwords won't work.
I'll update this post as soon as Microsoft follows up with me.
