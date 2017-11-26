---
title: "Github SSH Key Blackhole (single user key)"                                       
date: 2017-02-07
categories:                                         
  - security
  - code
tags:
  - test
thumbnailImagePosition: left
thumbnailImage: //d1u9biwaxjngwg.cloudfront.net/highlighted-code-showcase/peak-140.jpg
---

I was kind of dumbstruck at work when I got a permission error when a key was added. After realizing it was associated with another account, I realized that maybe there was a bigger problem since git uses a single user - "git" over ssh - for all normal command line transactions (there's also a REST API you can use to create repos and the like).

<!--more-->

So what's the issue? Well most systems use a username and "password" (password can be typed symbols or a public key or OTP token etc - it doesn't matter). The problem here is on most systems, there's a public part to a login (generally referred to as a username or id) and a private part - the "password".

Most systems do not let you use usernames for multiple entities (people, devices, etc) but do allow associating the same password with multiple usernames. Github works fine here because of a sane web interface, but partially fails with their ssh interface.

This is only half of what we need to know to mess with people here (or so I thought). The other bit is how ssh works. SSH is a first come, first serve tool. What I mean by this, is for the most part (there are a few exceptions that are addative), once an option is given to ssh or something works (like a key), ssh moves on - it doesn't look farther.

How does this help us? SSH will read through whatever is in the agent, whatever is listed under IdentityFiles option, and then whatever is under ~/.ssh/id_* (you can narrow this scope a bit with IdentitiesOnly, but this isn't default behavior). Which me if ssh tries a key that's in it's search path and it works for our user, it'll stop trying.

So I can add any key that I find on the internet to any user and it'll give them a different set of permissiions? Well no. This is what I thought at first, but it seems this is not the case. Bummer? Not quite. This leaves open two other possibilities:


1. I can still add keys that aren't already associated with users and essentially blackhole a key (they won't be able to associate it with their actual user). I don't need to verify ownership of a private key when adding a public key.

2. I can use this to test if a key is associated with a user. Which also means if there's a private key laying around, I just need to generate fingerprints off of all of the private keys I find and compare them with the public key that github isn't letting me add and I know there's some user on github I can own. Obviously I could also use the key with ssh and do, ssh -T git@github.com Verify and look for working keys that way, but I'm guessing github is looking for that more than public keys added to multiple users (maybe?).

Of note - Google isn't letting me find ssh public keys, but github found 120k. And searching duckduckgo for the string +"ssh-rsa AAAA" yields some results as well.
