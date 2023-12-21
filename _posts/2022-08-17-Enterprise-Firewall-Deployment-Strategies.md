---
pin: true
layout: post
title: "Enterprise Firewall Deployment Strategies"
date: 2022-08-17
author: shawn
categories:                                         
  - automation
tags:
  - security
  - idea
image:
  path: /static/img/The_Thinker_Rodin_Phila.jpeg
---

Modern systems deployments assume infrastructure as code (IaC) or basically that you can redeploy a system by making some parser read some file and make a duplicate setup to what you have or had. Deploying hardware from a tftp server which points to a code repo of configurations or deploying cloud VMs (EC2) from terraform are both models of this process.

There are also many firewall solutions: Security Groups, PF, IPTables, and all of the hardware vendors with their unique ways of doing things. I find the hardware solutions are managed ok or have their own religous way of doing things (and I prefer to stay out of others’ churches). PF seems to be the best firewall solution design, but it also has a limited audience. Security Groups are good at what they do, but have limitations (where one may use another cloud service to gap those - like ELBv2 for iptables’ prerouting or WAF for string matching) and I may want to try to write a conversion to handle this in the future, but not now. For now, I want to deploy an iptables firewall ruleset.

Before using Chef or Ansible or Salt or Puppet or CFEngine etc, most have probably written shell scripts to configure iptables. Maybe you even put logic in them so that production rules would only be enabled if the computer were in production? Or maybe even more complex logic to take a list of IP’s and apply some rule to each one of them (use ipset if you’re still doing this)? Then someone showed you a configuration management platform and you rewrote your logic for that platform. And that works right?

Most servers we deploy are for single applications that do a single thing. So in the beginning having a development environment for those applications with a pretty loose firewall ruleset works well enough. And then, when you push the application/server config to a production environment, you determine exactly what network access that application needs and put that application specific ruleset on top of your base rules. This works ok. You can even keep everything as simple manageable code by using “templates”, and that works well enough.

Note: when I say “templates” here, I’m thinking more of a bash template system vs jinja2 or ERB etc and more along the lines of bash that may look like:

```shell
source definitions.sh >/dev/null 2>&1
iptables_cmd=“/sbin/iptables”
for port in ${SOME_APP[@]:-}; do
  $iptables_cmd -A INPUT -p tcp —dport “$port” -j ACCEPT
done
```

The first problem you may notice is that it’s slow (I mean really slow). The last time I tried this with a shell script (admittedly a decade ago) 500 rules took about a minute to implement and this isn’t atomic. You flush rules, remove chains, and then repopulate. And while you’re doing this, you have no firewall. You’ve also put this in a service file, which also probably means it takes another minute to see if a reboot was successful. But one day, you discover iptables-save and iptables-restore and you get past this - kinda.

So we take our initial script and use it to lay out an iptables-save file like this:

```shell
source definitions.sh >/dev/null 2>&1
iptables_cmd=“echo —”
for port in ${SOME_APP[@]:-}; do
  $iptables_cmd -A INPUT -p tcp —dport “$port” -j ACCEPT
done
```

And redirecting that script’s output to a save file that you can load with iptables-restore < file finishes this migration process. And now, I’ve got my good enough solution that’s deploying in a development environment where devs can do their job and which deploys a tighter ruleset to production without too much trouble (I can’t really test my firewall in non-production where everything else is being developed, but that’s life). Cool.

But most environments use a configuration management solution to handle application deployment and configuration and iptables is a host based firewall solution that needs similar configuration. It is common to have a base firewall solution that may set a deny policy and a few permissive rules to allow most things to function transparently. This is, frankly, better than nothing - but getting groups to use a decently secure firewall without bumping up against limits and just disabling it is hard (similar to discussions about not disabling SELinux).

So then we’ve gone down the configuration management road and see that everyone has an iptables module/library. Except, they force a similar workflow as your old and slow shell script. And maybe that’s fine and you go with it and maybe it’s not so you template an iptables-save file and deploy that (the later being somewhat kludgy, but probably preferred). I think the better option is to work within your configuration management tool and create a system to easily deploy policies based on what we’re doing. Maybe this system could have variables (or just use a template system), but in the end, it should probably create a sane file but be simple enough for anyone to use at a high level.

So we’re no longer using bash to deploy (outside of docker/kubernetes/cloud-init). You write application specific plays which you want to use to also configure your host based firewalls. What this means is that we’ve kind of moved backward. If we use our configuration management solution to deploy 500 rules, it’s still going to take a minute or so extra because it’s just calling iptables each time you call that module - it is just an ETL for iptables that it executes tons of times under the hood. The only thing this conversion has bought you above your home rolled script is putting your firewall configuration along side your application deployment. This isn’t anything new though - you could always update your firewall as part of a post install in a deb or rpm package and not need a separate deployment for this.

Firewalld does has an interesting idea of using ‘service’ files. These files are xml which is fine - I can parse them, bulk modify, use xpath to get at specific information, etc. But the firewalld system also has issues with complexity: there’s no xsd (https://github.com/firewalld/firewalld/pull/492), helpers seem to be pretty opaque (https://firewalld.org/documentation/man-pages/firewalld.helper.html). Tying a server firewall solution directly into a “desktop bus” (dbus) seems needlessly bloated and to go against the Unix philosophy of doing one thing well (this does seem very different than fail2ban calling iptables while having a client/server architecture - https://firewalld.org/documentation/man-pages/firewalld.dbus.html). The last straw for me not liking firewalld was: in order to use any iptables-extension that’s not baked in (like doing a string match) needs a separate helper string match (which again, is opaque from the command line). An example of string matching in firewalld is: https://github.com/fusionpbx/fusionpbx-install.sh/blob/master/centos/resources/firewalld.sh. So if I want to use iptables/netfilter as a kernel level WAF, I need to create one helper per match and I can’t even report on them - not ok. All of those complaints aside, having a thing you can point to and say “I’m using this application/service that’s got a similar name to that of a property managed by a firewall framework” and then being able to say “yes, please enable it for me”, is smart.

Why Ansible? Frankly, because I wanted to learn python and ansible at the same time and someone had already created a module that I could work with (they made decently sane decisions). But really, I was also kinda hoping this was already done - I don’t like redoing work and had asked my former employer to open source my Chef LWRP Ruby work and had described it to Opscode/Chef people and asked them to implement this and neither have happened. And since I’ve done this in Chef (and bash and fairly decently in perl), I already have a pretty decent idea of how things should work around an enterprise iptables deployment. I know people need an on ramp if you want to push a new solution - so making a useful system that someone can choose to use or not with minimal pain is a must. I know rulesets become very proprietary very quickly, so those definitions should be very separate from the actual configuration utility. I’m also quite aware that most people (especially me) enjoys examples of how to use a new thing - which is more or less the point of me writing about this.

I plan to create a repo of popular rule sets that anyone can take, combine that json into one file, feed it into Ansible’s iptables module, and enable/disable rules at will. In order to do this, I needed to add some features to Ansible’s iptables module. This work is currently happening here: https://github.com/ansible/ansible/pull/78537 and while I’m currently considering enhancements (like passing vars with picker_definitions and allowing the definitions to be j2 templates and creating a better dependency system), the goal right now is to just get this pull request accepted basically as is so that others may use my rules (and hopefully other third party rules).

To facilite an easier on ramp (and maybe allow easier contributions to a public ruleset), I’ve also created an iptables parser. This may show up in an ansible galaxy repo at some point as it is intended as a fact collector (but not to generate a drop in structure for this ansible module) but is just a file for now: https://gist.github.com/sandboxcom/dbc1d949f879299a313d400dc5f7d990 which nests everything at the right level and names parameters correctly, but also scrapes out policies etc, so I didn’t think it should create the same structure. If you use this scraper, replace the table with the name of your policy (ie, ‘filter’ with ‘base’) and pass that in and:

```yaml
picker_includes:
  - base: true
```

But you probably want to split rules into application specific items too. I’ll have more to say on this later. These rule files are going to get really large really quickly and you should consider splitting them up into separate files and using something like jq to combine them for you. You may also use another data store and Ansible’s cache module to bring in corporate rules for you. I may not be documenting the corporate integration bit, but it should be pretty obvious how you’d implement something like that in a large environment.

