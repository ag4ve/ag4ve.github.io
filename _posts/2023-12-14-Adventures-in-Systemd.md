---
pin: true
layout: post
title: "Adventures in Systemd"
date: 2023-12-14
categories:                                         
  - automation
tags:
  - code
  - os
image:
  path: /static/img/systemd-logomark.svg
---

> Actions were done as root. Prompts are `$` instead of `#` to differenciate between comments in the output.
{: .prompt-info }

> I showed `systemctl cat` output instead of on disk file content to show runtime state of services.
{: .prompt-info }

# Kicking Around Systemd Events

The other day, I figured I'd automate my firewall setup a bit more by making it happen automatically. I try to avoid dependencies if I can and most Linux systems today run systemd. Unlike most of the internet, I don't have a strong opinion about systemd - I like that it's ubiquitous and I don't need to wonder what init system I'm writing for. So when I wanted to create a service for my firewall setup, I looked no further than systemd. This didn't end up being as obvious of a solution as I thought - but I did learn some things. I didn't see them talked about much, so I figured I'd write about them here.

## Drop-in

Primarily because I plan to use this feature in the rest of this post, lets cover the drop-in system of systemd. I think most people who have written systemd services before have put them in /etc/systemd/system, which is great. And if you want to know the full search path systemd uses, it's here [1]. But I saw lots of comments online that people didn't realize that you could make partial systemd unit files where you override parts of a larger unit file - these are called drop-in files. They're documented under the systemd.unit man page [2] under "Description".

Lets show how this works by first creating a basic unit. I'm going to do this in /etc/systemd (because that's where I'm supposed to do this) but this would work the same in any other directory in the search path). First lets see what systemd sees as our initial test service file:

```console
$ systemctl cat test
# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test http://www.example.com

[Service]
ExecStart=echo "systemd testing"
```

And then, create the directory for our drop-in, create a drop-in file in that directory, and see what systemd sees:

```console
$ mkdir test.service.d
$ echo -e '[Service]\nExecStart=echo "systemd testing 2"' > test.service.d/00-echo.conf
$ systemctl cat test
# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test http://www.example.com

[Service]
ExecStart=echo "systemd testing"


# /etc/systemd/system/test.service.d/00-echo.conf
[Service]
ExecStart=echo "systemd testing 2"
```

And finally, lets run it and see look at the echo output (the rest is too noisy - try it yourself):

```console
$ systemctl start test
$ journalctl -xe -u test --no-pager | grep echo
Dec 11 02:30:28 srwilson-u2204 echo[3582579]: systemd testing
Dec 11 02:30:28 srwilson-u2204 echo[3582580]: systemd testing 2
```

As you can see, running the test service shows output from both the original service and the drop-in.

## Templates

Have you ever wanted to pass parameters to your services? Maybe an IP address or username or network card? This is also documented under systemd.unit Description [2]. Prefix your service with an @, such as test@.service and you'll be able to do just that:

```console
$ systemd-escape --template=test@.service "foo bar baz"
test@foo\x20bar\x20baz.service
$ systemctl start "$(systemd-escape --template=test@.service "foo bar baz")"
# journalctl -xe -u 'test@*' --no-pager | grep echo
Dec 11 02:46:05 srwilson-u2204 echo[3595571]: systemd testing raw foo\x20bar\x20baz
Dec 11 02:46:05 srwilson-u2204 echo[3595572]: systemd testing foo bar baz
```

And if you look at how your system is currently using these templates, you'll find some interesting use cases and notice the parameters have been placed in the documentation section too:

```console
$ systemctl list-units | grep @
  getty@tty1.service                                                                        loaded active     running   Getty on tty1
  getty@tty2.service                                                                        loaded active     running   Getty on tty2
  lvm2-pvscan@253:0.service                                                                 loaded active     exited    LVM event activation on device 253:0
  systemd-cryptsetup@dm_crypt\x2d0.service                                                  loaded active     exited    Cryptography Setup for dm_crypt-0
  systemd-fsck@dev-disk-by\x2duuid-6d46f500\x2db876\x2d4dd5\x2d901a\x2dcd04a4526a7d.service loaded active     exited    File System Check on /dev/disk/by-uuid/6d46f500-b876-4dd5-901a-cd04a4526a7d
  systemd-fsck@dev-disk-by\x2duuid-6F53\x2dCD7B.service                                     loaded active     exited    File System Check on /dev/disk/by-uuid/6F53-CD7B
  user-runtime-dir@0.service                                                                loaded active     exited    User Runtime Directory /run/user/0
  user-runtime-dir@1000.service                                                             loaded active     exited    User Runtime Directory /run/user/1000
  user@0.service                                                                            loaded active     running   User Manager for UID 0
  user@1000.service                                                                         loaded active     running   User Manager for UID 1000
  blockdev@dev-mapper-dm_crypt\x2d0.target                                                  loaded active     active    Block Device Preparation for /dev/mapper/dm_crypt-0
```

# Other systemd apps

Half of the programs systemd comes with aren't in /usr/bin, but in /usr/lib/systemd:

```console
$ find /usr -type f -perm /111 \( -iwholename "/usr/lib/systemd/*" -o -iwholename "/usr/bin/*" \) -iname "systemd-*" | column
```

Things like systemd-networkd-wait-online (SNWO) which has a service wrapper for it so other units aren't started before you're online. Ironically, I'm currently using netplan to connect to wifi (I'm most definitely online) and SNWO says it's failed - SYSTEMD_LOG_LEVEL=debug ./systemd-networkd-wait-online doesn't have my wifi card listed. There's also systemd-nspawn (on Ubuntu, I had to install systemd-container to get) which will start a service in a namespace (container), systemd-cgtop which is a top like tool for cgroups, 

## Debugging

Notice that most (all?) of the executables in /lib/systemd seem to give extra information with a SYSTEMD_LOG_LEVEL=debug? Since we see that and that lots of these programs are wrapped in services with:

```console
$ find -type f -perm /111 | while read f; do echo "${f##*/}"; done | while read f; do [[ "$f" == "systemd" ]] && continue; echo $f; grep -rE "Exec.*=.*$f"; done
```

We know we can create `/etc/systemd/system/<unit>.service.d/00-debug.conf` that look like:

```shell
[Service]
Environment=SYSTEMD_LOG_LEVEL=debug
```

And get debug information from any of these services. And most programs take a variable to enable debugging. In a worst case, the linker has LD_DEBUG=all that you can use for any dynamic C programs. And this output would show up in the journal log. Lastly, systemd relies heavily on dbus and there's a great blog about that here [3].

## My journey

Now lets get back to why I started looking into this. I've written systemd services before and don't have a need to stand up a tomcat server. I came here to manage a network parts of my network after a connection was established. Which means, I want a systemd service to trigger on some event. There are a few ways to get an event to happen in systemd: a timer, path, udev, or relationship to another service. I'm pretty sure that's it. Obviously a timer doesn't do me any good in this situation, so we'll consider the other three.

First lets deviate into the network specific options systemd does have because it's quite alot. The systemd.link options deals with your hardware and has documented overlap with udev (and also seems to overlap with ethtool options). But basically anything you could wish to do with your network hardware is probably there. The systemd.network configures the network for those devices. And then systemd.netdev configures virtual networks. But none of these unit files have the option to then kick off a service, so I moved on.

### UDEV - unsuccessful

The udev filesystem has been around for almost 20 years now - it quickly took over for devfs in the early 2000s. When a devices is plugged in that matches a udev rule, you can have it run a service by having this with the line that matches your device:

```console
RUN="systemctl --no-block start <service>"
```

And you may want to do this to load the new rule and get already plugged in devices run through the rule:

```console
$ udevadm control --reload-rules
$ udevadm trigger
```

But this is moot. Network links aren't hardware. I don't remove my nic (or even adjust the power) when getting online or disconnecting. So the rule never fires. Next option.

### Tailing - successful

Instead, we can look at something happening in a file. If that thing shows up, we can do something and restart:

```console
$ systemctl cat test
# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test http://www.example.com

[Service]
Type=simple
Restart=on-success
ExecStart=sh -c 'tail -n1 -f /var/log/syslog 2>/dev/null | grep -q -m1 foobar'
ExecStopPost=echo "systemd testing"
```

The simplest way to test this is with the logger command:

```console
$ logger foobar
```

Which then matches and we'll see the ExecStopPost message. We can also use After and Requires to kick off another service from our event:

```console
$ systemctl cat test*
# /etc/systemd/system/test2.service
[Unit]
Description=testing 2
Documentation=test2 http://www.example.com

[Service]
Type=simple
ExecStart=echo test2


# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test http://www.example.com
After=test2.service
Requires=test2.service
 
[Service]
Type=simple
Restart=on-success
ExecStart=sh -c 'tail -n1 -f /var/log/syslog 2>/dev/null | grep -q -m1 foobar'
```

Which we can then fire off with the same logger command and see getting processed by looking at the journalctl:

```console
$ journalctl -xe -u test2 | grep echo | wc -l
```

We can also reverse the dependency structure by putting RequiredBy in an [Install] section and enabling the test2 service. However, in order to ensure the test service is running by starting test, we need to use BindsTo or Wants 

```console
$ systemctl cat test2
# /etc/systemd/system/test2.service
[Unit]
Description=testing 2
Documentation=test2 http://www.example.com
BindsTo=test.service

[Service]
Type=simple
ExecStart=echo test2

[Install]
RequiredBy=test.service
$ systemctl enable test2
Created symlink /etc/systemd/system/test.service.requires/test2.service → /etc/systemd/system/test2.service.
$ systemctl start test2
$ systemctl is-active test
active
```

Which means that we only need to know about test2 and not ensure test is running (it's handled for us). But test2 runs twice or:

```console
$ systemctl cat test2
# /etc/systemd/system/test2.service
[Unit]
Description=testing 2
Documentation=test2 http://www.example.com

[Service]
Type=simple
ExecStart=echo test2

[Install]
RequiredBy=test.service
$ systemctl enable test2
Created symlink /etc/systemd/system/test.service.requires/test2.service → /etc/systemd/system/test2.service.
$ systemctl start test
```

And we need to manage the service that fires events for us but events are handled properly and we only get one event. See [4] For a logic diagram of Wants, PartOf, and Requires see [4].and there's a mapping of associative properties and inverses here [5].

### Path

And finally, we can create a path file which monitors paths like:

```console
$ systemctl cat test.path
# /etc/systemd/system/test.path
[Unit]
Description=testing
Documentation=test http://www.example.com

[Path]
PathModified=/root/foo
Unit=test2.service

[Install]
WantedBy=multi-user.target
```

Which does as it's intended to and test2 kicks off when /root/foo is modified. However, when I try to do: PathModified=/sys/devices/virtual/net/exttest0/operstate nothing happens. The reason for this is because sysfs isn't an actual filesystem and just an interface to kernel memory.

## Conclusion

It seems a bit overly complex to have an event based service. Needing to enable a service in order to install reverse dependencies seems a bit unnecessary. However, the service runner does a pretty good job for most cases. I do enjoy not needing to write full wrapper scripts for my services anymore. And for this task, having a service that does:

```console
ExecStart=sh -c 'ip monitor link | grep ",UP,LOWER_UP"'
```

And relying on that in another service should do what I need well enough.

## Links

1. [https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Unit%20File%20Load%20Path](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Unit%20File%20Load%20Path)
2. [https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)
3. [https://0pointer.net/blog/the-new-sd-bus-api-of-systemd.html](https://0pointer.net/blog/the-new-sd-bus-api-of-systemd.html)
4. [https://pychao.com/2021/02/24/difference-between-partof-and-bindsto-in-a-systemd-unit/](https://pychao.com/2021/02/24/difference-between-partof-and-bindsto-in-a-systemd-unit/)
5. [https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Mapping%20of%20unit%20properties%20to%20their%20inverses](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Mapping%20of%20unit%20properties%20to%20their%20inverses)

[1]: https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Unit%20File%20Load%20Path
[2]: https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html
[3]: https://0pointer.net/blog/the-new-sd-bus-api-of-systemd.html
[4]: https://pychao.com/2021/02/24/difference-between-partof-and-bindsto-in-a-systemd-unit/
[5]: https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Mapping%20of%20unit%20properties%20to%20their%20inverses

