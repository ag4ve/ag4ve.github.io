---
title: "Finding a host"                                       
date: 2017-10-22
categories:                                         
  - osint
  - network
tags:
  - highlight code
  - github theme
  - test
thumbnailImagePosition: left
thumbnailImage: //d1u9biwaxjngwg.cloudfront.net/highlighted-code-showcase/peak-140.jpg
---

I figured I needed to get back on IRC today, went to login and got this message:

<!--more-->

{{< codeblock "irc output" "bash" "http://underscorejs.org/#compact" "irc.out" >}}
08:58 [Freenode] -NickServ(NickServ@services.)- This nickname is registered. Please choose a different nickname, or identify via /msg NickServ identify <password>.
08:58 [Freenode] -!- Irssi: Your nick is owned by Got ZNC? [~user@ns3301026.ip-5-135-157.eu]
{{< /codeblock >}}

Ok, that sucks, but I do remember trying to some free znc servers (irc bouncer/relay) a while back - different topic - google it. Point being, I don't remember which one I might've gone with because they had a waiting period and I moved on to other things before that went online (obviously). So now I need to figure out what I used.

The server that is logged on with my username is ns3301026.ip-5-135-157.eu

Of interest - it's subdomain of some domain that is for a dynamic ip or similar (ie, not a useful name to me). Just for the heck of it, I did a whois on it, and in part, got this:

{{< codeblock "whois" "bash" "http://underscorejs.org/#compact" "whois.out" >}}
Domain: ip-5-135-157.eu

Registrant:
        NOT DISCLOSED!
        Visit www.eurid.eu for webbased whois.

Onsite(s):
        NOT DISCLOSED!
        Visit www.eurid.eu for webbased whois.

Technical:
        Name: Klaba Octave
        Organisation: OVH
        Language: fr
{{< /codeblock >}}

I get all warm and fuzzy when I see something attached to me being used on an OVH server (big European datacenter, but also where lots of bad stuff comes out of - pretty sure Equinix or Verizon doesn't do similar in Europe). But I digress.

So I still don't know who owns that host. I can also do a whois of the IP and if the person has an ASN, I can tell that (but due to it being hosted, I'm not too hopeful here).

{{< codeblock "ip whois" "bash" "http://underscorejs.org/#compact" "ip-whois.out" >}}
route:          5.135.0.0/16
descr:          OVH
origin:         AS16276
mnt-by:         OVH-MNT
created:        2012-07-06T13:00:08Z
last-modified:  2012-07-06T13:00:08Z
source:         RIPE # Filtered
{{< /codeblock >}}

And indeed millions of IPs there (the /16) belong to OVH - literally nothing more to see here)).

What I can also do is reverse name lookup for the /24 (could do it for the /16, but that might still be going on as I write this and is kinda wasteful since I don't care). Actually, I could've just done 160-200, but the /24 is fine. Out of that, I see:

{{< codeblock "bash-loop.out" "bash" "http://underscorejs.org/#compact" "bash-loop.out" >}}
for i in {1..254}; do domain="$(dig -x 5.135.157.$i +short)"; [[ -n "$domain" && "$domain" != *.ip-* ]] && echo "$i: $domain" ; done

And around 180, we see:
171: ks3301017.kimsufi.com.
173: ks3301019.kimsufi.com.
176: fremini560.xirvik.com.
177: mail1.edan.fr.
179: server.androx.ovh.
182: ks3301028.kimsufi.com.
185: ks3301031.kimsufi.com.
188: ks3301034.kimsufi.com.
198: ks3301734.kimsufi.com.
199: ks3301735.kimsufi.com.
201: ks3301737.kimsufi.com.
{{< /codeblock >}}

So I'm guessing there's some company called kimsufi or edan (saw that in the whois) that I'm looking for. Now to google "free znc host" and see what pops out.

