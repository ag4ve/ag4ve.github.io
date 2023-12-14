---
pin: true
layout: post
title: "NFT - Linux Extensible Firewall"
date: 2023-12-12
categories:                                         
  - automation
tags:
  - security
  - code
  - os
image:
  path: /static/img/The_Dark_Side_of_the_Moon_Cover.svg
---

We’re discussing Linux firewalls. As such this post is technical. While I'm trying to show how to write a modular firewall system vs go into detail about networking, it does help to have some background knowledge on how firewalls work, because I'm not covering that here.

The code behind this blog is at: [https://github.com/ag4ve/nft-policy](https://github.com/ag4ve/nft-policy) [1] and is signed with a key with a fingerprint of: BD8A 77EE 8991 9B51 5D73 E795 B8AB 96D5 2BF0 B6D4 and you can also look it up on github or keybase.

## Preface

I don't setup a NAT (network address translation) every day, so I figured I'd do a quick Google on the best way to do that. One of the solutions I stumbled on showed nft commands. I knew that nftables had been under development for a while but hadn't considered that it may be installed on my Ubuntu LTS system out of the box. The more I read the more interested I was in the improvements nftables had over the older iptables - given the features nftables present, I should really say: I'm impressed with nft over xtables-multi, but I digress.

So let’s step back a bit. Over the years, Linux has had a few firewall systems: ipfirewall, ipchains, iptables, and now nftables. Some may also point at FirewallD [3] or UFW [4] (or others?) - they're frontends though. In the past few years eBPF (and now, just BPF) has started taking over networking and debugging functions in Linux. There are projects like Cilium that have taken over large parts of the firewall/router functionality for Linux primarly with Kubernetes. However, when using Linux as a platform in of itself, you'll find either iptables or nft are what you interact with for routing/firewall features.

With that background, out of the way, I want to talk about using the features of nftables (the nft command) to create an extensive firewall system. I used "NFT" as the title hoping some poor soul would mistake this post as being about non-fungible tokens.

## Features

While there are many cool features nftables has that iptables do not, there are two main features that will help you make an extensible firewall using nftables on whatever system you're configuring: includes and variables. 

Consider what you need to think about when creating a firewall/router - probably an application or project and what it’s trying to do on the network, right? And if you've been doing this for a while, you make a backup of what you currently have, copy that backup to <project name>.ipt and go to work updating it to suit your needs at the moment. But what if you have 10 or 100 or 1000 servers that you want to deploy some applications on? They all have some standard things they do: allow ssh out and/or in, get a dhcp lease, talk to a dns server, sync time, maybe other things - but a fairly limited set of common network tasks, right? Some subset of those servers may be web servers, or database servers, or ldap servers, etc - they need separate rules for those services. Do you maintain per server rules? Do you maintain separate rules and merge them all together somehow? Or do you keep a loose policy and just not worry about it? In my experience, most people choose the latter option.

In our environment, most servers do the same things most of the time, but when there's a change, it impacts a number of rules. What if we could just apply a variable for each server or service or anything else that may change and just change that variable? When reloading, a variable can impact the ruleset, you don't even need to know what's happening in the ruleset to have lots of power.

## Housekeeping

### Backups

Before you begin tearing stuff apart, let’s make sure we have backups. In 2023, there are probably two firewalls on your computer: iptables and nftables - you should look for and backup both:
```console
# iptables-save > ~/iptables.save
```

This may also be called iptables-legacy-save. 

Then backup nftables:
```console
# nft list ruleset > ~/base.nft
```

You may not have any rules defined, but you will if you run docker or other container or virtual environments. It's best to backup nothing than lose a working system.

### Comparing changes

Both with iptables and nft, it's useful to see how we can compare what was there with the changes we've made. With nftables, it looks like this:
```console
# diff -U0 base.nft <(nft list ruleset)
```

With iptables, it would be:
```console
# diff -U0 iptables.save <(iptables-save)
```

If you load the save file you created, it may create a slightly different running config. So, you're going to want to compare what you have running and what is saved and reload the config from your save file and compare again to ensure they're basically the same. For example, the running state or backup may contain extra spaces, different counter stats, or ordering of what's in the rule (the order of statements in a rule actually matters with nftables but having different order was common with iptables). After you get a backup that gives a minimal diff with what's running, feel free to move on (this comparison took me a good 10 minutes).

### Changing systems

It's widely documented that using iptables and nftables at the same time may cause unpredictable behavior so you'll want to disable iptables if you decide to move forward with nftables. On Ubuntu/Debian this means disabling ufw, and on Redhat/Fedora this means disabling firewalld. You may want to do this last, after you have your new firewall in place, but you'll want to look at your current configuration, so might as well look at what's currently running. On Ubuntu, I do:

```console
# systemctl stop ufw
# systemctl disable ufw
Synchronizing state of ufw.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable ufw
Removed /etc/systemd/system/multi-user.target.wants/ufw.service.
```

And then:

```console
# systemctl start nftables
# systemctl enable nftables
Created symlink /etc/systemd/system/sysinit.target.wants/nftables.service → /lib/systemd/system/nftables.service.
```

## Beginnings

I want a script to load up and run all of my nftables scripts. Let’s create that and include our base.nft file in it:

```console
# cat nftables.nft 
#!/usr/sbin/nft -f

flush ruleset

include "./base.nft"

# chmod +rwx nftables.nft
```

We should be able to run ./nftables.nft and do a diff with the base.nft and the ruleset and shouldn't see many changes. You should be able to run that nftables script and then do that diff all day with no or minimal output (differences). If you're still confident with what you have running and your backup, let’s move on.

After we start creating per service policy files, the diff against base won't look clean - you'll see the service rules you've created listed. What you can do at that point is run a:
```console
# nft list ruleset > snapshot.nft
```

At some point, you'll know what you had running, which you can compare against the running config.

## First service

What I suggest is that you look through your base.nft and find something that's not listed that you want to write a rule for and start with that. On my system, I only see docker and qemu rules, so I want to start with http and https egress. Maybe I can log IPs and how much data that I've sent to each server? I'm going to call the file for these rules, http_out.nft:

```console
# cat vars.nft 
define ext_if = "wip3s0"
# cat http_out.nft 
table ip filter {
        set http_hosts {
                type ipv4_addr;
                flags dynamic;
                size 65536;
                timeout 60m;
        }
        chain OUTPUT {
                type filter hook output priority filter; policy accept;
                ct state new tcp dport {http,https} update @http_hosts {ip daddr counter}
                iifname $ext_if tcp dport {http,https} counter packets 0 bytes 0 accept
        }
}
```

Lets look for the http/https host I'm the most talkative with:

```console
root@srwilson-u2204:~# nft --json list set filter http_hosts \
  | jq -rSC '
    .nftables[] 
      | select(.set) 
      | .set.elem[].elem 
      | [.counter.packets, .val] 
      | @tsv' \
  | sort -nr \
  | head
42      74.121.143.245
42      74.121.143.240
30      192.208.222.110
22      54.183.214.39
16      8.39.36.141
14      8.39.36.142
14      54.241.73.82
14      35.186.154.107
13      54.151.69.61
12      54.215.155.110
```

And of course, there's more fun to be had with a list like this:

```console
# nft --json list set filter http_hosts \
  | jq -rSC '
    .nftables[] 
      | select(.set) 
      | .set.elem[].elem 
      | [.counter.packets, .val] 
      | @tsv' \
  | sort -nr \
  | head \
  | while read _ ip; do \
    domain="$(dig -x $ip +short)"; \
    echo -e "$ip \t $domain"; \
  done
74.121.143.245
74.121.143.240
192.208.222.110
54.183.214.39    ec2-54-183-214-39.us-west-1.compute.amazonaws.com.
8.39.36.142
8.39.36.141
54.241.73.82     ec2-54-241-73-82.us-west-1.compute.amazonaws.com.
35.186.154.107   107.154.186.35.bc.googleusercontent.com.
54.151.69.61     ec2-54-151-69-61.us-west-1.compute.amazonaws.com.
54.215.155.110   ec2-54-215-155-110.us-west-1.compute.amazonaws.com.
```

You'll also notice that I created a variables file for my external network card. So now, no matter if you're using eth0, en0, or anything else, you can still use that policy file - and any other policy files that use that variable just by changing that one variable in that file. My nftables.nft script now looks like:

```console
# cat nftables.nft 
#!/usr/sbin/nft -f

flush ruleset

include "./vars.nft"
include "./base.nft"
include "./http_out.nft"
```

## Debugging

When we write rules, we're going to find that some protocols just don't do what we expect. Even if you carefully read the RFC for the protocol you're creating a rule for, you'll find that the RFCs have definitions for "may" and "may not" - implying that the protocol can deviate from what is stated and be a completely valid protocol. Having said that, there are tons of protocols that are broken. And then there are proprietary protocols that don't have RFCs at all. Reading an RFC to create a single rule that may not work is time consuming for minimal payoff, so why bother? Instead, I recommend making a best effort rule and refining it.

In that vein, I created a dns_out.nft rule:

```console
# cat dns_out.nft 
table ip filter {
        chain out_dns_drop {
                limit rate 100/minute burst 150 packets \
                        log flags all prefix "$log_prefix Bad DNS server: " \
                        drop
        }
        chain OUTPUT {
                type filter hook output priority filter; policy $filter_out_policy;
                iifname $out_if meta l4proto {tcp,udp} th dport 53 ip daddr $dns_servers counter packets 0 bytes 0 accept
                meta l4proto {tcp,udp} th dport 53 jump out_dns_drop
        }
}
```

You'll notice that I created a chain that I can jump to that logs and drops DNS packets if the prior rule didn't accept them. I had populated $dns_servers in my vars.nft file with dns servers I thought I might use, but even after looking at netstat -ntap to help populate my list, I forgot one:

```console
# grep 'NFT Bad DNS server:' /var/log/syslog
Dec  6 09:22:53 srwilson-u2204 kernel: [1784699.108298] NFT Bad DNS server: IN= OUT=wlp3s0 SRC=192.168.4.50 DST=192.168.4.1 LEN=73 TOS=0x00 PREC=0x00 TTL=64 ID=30442 PROTO=UDP SPT=49043 DPT=53 LEN=53 UID=102 GID=103
```

And so, my home router is providing a DNS server for me – and the computer is  probably configured through DHCP as I didn't find it in /etc/systemd/resolved.conf or /etc/resolv.conf files. And so, my variable looks like:

```ruby
define dns_servers = {
        192.168.4.1,
        127.0.0.53,
        8.8.8.8,
        8.8.4.4
}
```

## Handling ICMP

ICMP is generally good and if you blindly drop these packets, you're going to break things (i.e., IPv6 won't work at all) and you may not like your decision if you need to troubleshoot your network. So let’s consider a nice policy for this. To better document our rule, we can use protocol names for lots of what we want to do here. We look up these names using the describe option for nft, such as:

```console
# nft describe icmp code
payload expression, datatype icmp_code (icmp code) (basetype integer), 8 bits

pre-defined symbolic constants (in decimal):
        net-unreachable                                    0
        host-unreachable                                   1
        prot-unreachable                                   2
        port-unreachable                                   3
        net-prohibited                                     9
        host-prohibited                                   10
        admin-prohibited                                  13
        frag-needed                                        4
```

We can also "describe icmpv6 code". These rules ended up being over 70 lines and are incomplete. They're called icmp_in.nft in the Github repo for this project.

## Tracing

First a note on the order of things. When we include these rulesets, they're inserted in the order they're included. So, vars.nft comes first (so that other rules can reference those variables) and then in_checks.nft (well, I have the base.nft I started before that, but the idea is to remove or minimize that - I also have git ignoring base.nft). We can then define a trace rule, but we should try to define it sooner than later so that we can catch more data when it's turned on. As such, I defined two trace rules:

```console
# grep -r nft_trace
in_checks.nft:          meta nftrace set $nft_trace
out_checks.nft:         meta nftrace set $nft_trace
vars.nft:define nft_trace = "1"
```

This means I’ll see everything when I enable the nft_trace variable and run my script to reload the firewall policy:

```console
# nft monitor trace | tee nft.trace
```

You should be able to run nft -j monitor in order to get json output, but that doesn't seem to work for me. There's a hash/id per packet that is routed through the system and we can see what is happening to it. Since the firewall doesn't see just one packet at a time, I picked a hash to look at:

```console
# grep d3a8f023 nft.trace
trace id d3a8f023 ip filter OUTPUT packet: oif "wlp3s0" ip saddr 192.168.4.50 ip daddr 192.168.4.30 ip dscp cs0 ip ecn not-ect ip ttl 64 ip id 17390 ip length 52 tcp sport 53064 tcp dport 8009 tcp flags == ack tcp window 795
trace id d3a8f023 ip filter OUTPUT rule meta nftrace set 1 (verdict continue)
trace id d3a8f023 ip filter OUTPUT rule ct state established,related counter packets 381 bytes 54097 accept comment "Permit established/related connections" (verdict accept)
trace id d3a8f023 ip mangle POSTROUTING packet: oif "wlp3s0" ip saddr 192.168.4.50 ip daddr 192.168.4.30 ip dscp cs0 ip ecn not-ect ip ttl 64 ip id 17390 ip length 52 tcp sport 53064 tcp dport 8009 tcp flags == ack tcp window 795
trace id d3a8f023 ip mangle POSTROUTING verdict continue
trace id d3a8f023 ip mangle POSTROUTING policy accept
```

This shows my computer sending a packet (in acknowledgement) to another host on my network. Each line starts with 'trace id' followed by the ID, and then the type of chain, the hook, the chain name, and then what rule processed the packet. It's a much nicer system for trying to determine what is happening than looking at counters.

## Auditing

Many years ago, I worked for a small pentesting company. There were times we were looking for a solution to audit iptables rules. A few years ago, I did vulnerability management and used a product called Nipper [9]. I've also seen Nessus try to report on iptables, but it seems to just tell you if you have an "allow" policy. However, nftables can output rules in json format and OPA [5] allows auditing json files. There's tons of documentation on OPA, so I'm not going to cover it here, but I have started writing some tests that I'll add to the nft-policy repo.

We can look at the above trace (and other traces) to come up with real world tests. I've found the trace helps me see like a packet and then I can say: if I were a packet being sent out, I'd go through the output handler, that has an OUTPUT chain hooked onto it, and as long as nothing else drops or rejects me first, I'm going to hit the ct rule and get accepted. Each step there should be a test. I haven't turned the policy into a firewall as you can see from the last line of the above trace, the packet gets through because the mangle POSTROUTING chain has an accept policy - that should be tested for too and that test should currently fail - we want all policies to be drops.

## Backward compatibility

There are some applications that still want to use iptables. Mainly, virtual machine and container systems. The Archlinux wiki [7] has the best solution for this: use a netvm. They've got a systemd service file for making this happen for docker under the troubleshooting section at the end of their wiki.

## Last bits

I'm new to nftables, so I might be missing a bit. In fact, most of the rules were taken from questions or documentation or posts. I've considered whether it may be better practice to create multiple chains with different priorities and point them to the same hook in order to determine precedence - only time (or someone yelling at me) will tell. But what I do believe is that this system allows for a community ruleset where we create protocol or project-based rules that someone can just include and use like any other packaging system. Granted, including variables like I am doing creates a global scope and there is no way to scope chain names, so this will never be like any programming module system. But if we create a style guide (especially for naming), a system like this could work fairly well. If you plan to use a deployment system (i.e., Ansible) to deploy a system like this, the only templates should be the vars and nftables.nft file. You'd then deploy policy files and include them and modify the vars, as needed. I may create an Ansible role and Chef cookbook to demonstrate this, because it's a simple idea and doesn't have many moving parts.

I hope we adopt some modular firewall policy system. I attempted to start a system like this years ago [2], but there were issues with it, and I abandoned it. I created a Chef LWRP that deployed iptables policies based on service definitions at a large organization and that seemed to work. So maybe this will too? Let me know your thoughts. 

## Links

0. [https://wiki.nftables.org/wiki-nftables/index.php/Main_Page](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)
1. [https://github.com/ag4ve/nft-policy](https://github.com/ag4ve/nft-policy)
2. [https://github.com/ag4ve/NF-Save/blob/master/examples/policy.yml](https://github.com/ag4ve/NF-Save/blob/master/examples/policy.yml)
3. [https://firewalld.org](https://firewalld.org)
4. [https://wiki.ubuntu.com/UncomplicatedFirewall](https://wiki.ubuntu.com/UncomplicatedFirewall)
5. [https://www.openpolicyagent.org](https://www.openpolicyagent.org)
6. [https://wiki.gentoo.org/wiki/Nftables](https://wiki.gentoo.org/wiki/Nftables)
7. [https://wiki.archlinux.org/title/nftables](https://wiki.archlinux.org/title/nftables)
8. [https://www.firezone.dev/docs/reference/firewall-templates/nftables](https://www.firezone.dev/docs/reference/firewall-templates/nftables)
9. [https://www.titania.com/products/nipper](https://www.titania.com/products/nipper)

[0]: https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
[1]: https://github.com/ag4ve/nft-policy
[2]: https://github.com/ag4ve/NF-Save/blob/master/examples/policy.yml
[3]: https://firewalld.org
[4]: https://wiki.ubuntu.com/UncomplicatedFirewall
[5]: https://www.openpolicyagent.org
[6]: https://wiki.gentoo.org/wiki/Nftables
[7]: https://wiki.archlinux.org/title/nftables
[8]: https://www.firezone.dev/docs/reference/firewall-templates/nftables
[9]: https://www.titania.com/products/nipper
