---
pin: true
layout: post
title: "Adventures in Systemd"
date: 2023-12-14
author: shawn
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

Primarily because I plan to use this feature in the rest of this post, let's cover the drop-in system of systemd. I think most people who have written systemd services before have put them in `/etc/systemd/system`, which is great. If you want to know the full search path systemd uses, it's here [1], but I saw lots of comments online that people didn't realize that partial systemd unit files could be made to override parts of a larger unit file - these are called drop-in files. They're documented under the systemd.unit man page [2] under "Description".

The first thing to do to see how this works is to create a basic unit. I do this in `/etc/systemd/system` (because that's where it's supposed to be done) but this would work the same in any other directory in the search path. Systemd sees our initial test service file as:

```console
$ systemctl cat test
# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test https://www.example.com

[Service]
ExecStart=echo "systemd testing"
```

Second, create the directory for the drop-in and create a drop-in file in the `/etc/systemd/system` directory. Systemd sees this as:

```console
$ mkdir test.service.d
$ echo -e '[Service]\nExecStart=echo "systemd testing 2"' > test.service.d/00-echo.conf
$ systemctl cat test
# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test https://www.example.com

[Service]
ExecStart=echo "systemd testing"


# /etc/systemd/system/test.service.d/00-echo.conf
[Service]
ExecStart=echo "systemd testing 2"
```

Finally, let's run it and look at the echo output (the full output is noisy - try it yourself):

```console
$ systemctl start test
$ journalctl -xe -u test --no-pager | grep echo
Dec 11 02:30:28 srwilson-u2204 echo[3582579]: systemd testing
Dec 11 02:30:28 srwilson-u2204 echo[3582580]: systemd testing 2
```

As you can see, running the test service shows output from both the original service and the drop-in.

## Templates

Have you ever wanted to pass parameters to your services? Maybe an IP address or username or network card? This is also documented under systemd.unit Description [2]. Prefix your service with an `@`, such as `test@.service` and you'll be able to do just that:

```console
$ systemd-escape --template=test@.service "foo bar baz"
test@foo\x20bar\x20baz.service
$ systemctl start "$(systemd-escape --template=test@.service "foo bar baz")"
# journalctl -xe -u 'test@*' --no-pager | grep echo
Dec 11 02:46:05 srwilson-u2204 echo[3595571]: systemd testing raw foo\x20bar\x20baz
Dec 11 02:46:05 srwilson-u2204 echo[3595572]: systemd testing foo bar baz
```

If you look at how your system is currently using these templates, you'll find some interesting use cases and notice the parameters have been placed in the documentation section too:

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

Half of the programs systemd comes with aren't in `/usr/bin`, but in `/usr/lib/systemd`. I wouldn't add this to my PATH, but there's some useful programs we can use in our systemd units that we should be aware of:

```console
$ find /usr -type f -perm /111 \
  \( -iwholename "/usr/lib/systemd/*" -o -iwholename "/usr/bin/*" \) \
  -iname "systemd-*" \
  | column
```

There are programs like systemd-networkd-wait-online (SNWO) which have a service wrapper so other units aren't started before you're online. Ironically, I'm currently using netplan to connect to wifi (I'm most definitely online) and SNWO says it's failed - `SYSTEMD_LOG_LEVEL=debug ./systemd-networkd-wait-online` doesn't list my wifi card. There's also a systemd-nspawn command (on Ubuntu, I had to install systemd-container to get it) that will start a service in a namespace (container), systemd-cgtop, which is a top like tool for cgroups.

## Debugging

As mentioned in the last section, using `SYSTEMD_LOG_LEVEL=debug` with the commands in `/lib/systemd` provides quite a bit more information about what is happening with systemd commands. As shown below, lots of these commands are wrapped in services:

```console
$ find -type f -perm /111 \
  | while read f; do \
    echo "${f##*/}"; \
  done \
    | while read f; do \
      [[ "$f" == "systemd" ]] && continue; \
      echo $f; \
      grep -rE "Exec.*=.*$f"; \
    done
```

I demonstrated drop-ins earlier - when used with the `SYSTEMD_LOG_LEVEL`, we can easily get debug messages from any of these services. Create `/etc/systemd/system/<name>.service.d/00-debug.conf` that has:

```shell
[Service]
Environment=SYSTEMD_LOG_LEVEL=debug
```

Reload and restart the service then you'll get debug information from that service. Most other programs take an environment variable to enable some debugging information, too, which can be used similarly. In a worst case, when using dynamic programs, the linker provides `LD_DEBUG=all`. This output would show up in the journal log of your service. Lastly, systemd relies heavily on dbus and there's a great blog about using that here [3].

## My journey

Now let's get back to why I started looking into this. I've written systemd services before and don't have a need to stand up a tomcat server. I came here to manage the firewall and route parts of my network after a connection was established. This means, I want a systemd service to trigger on some event. There are a few ways to get an event to trigger systemd: a timer, path, udev, or relationship to another service. I'm pretty sure that's it. A timer doesn't do me any good in this situation, so we'll consider the other three.

First systemd has many network options and a configuration type for the lower layers of the OSI model. The systemd.link options deal with your hardware link (but not hardware - OSI layer 2) and have documented overlap with udev (and also seems to overlap with ethtool options). Basically anything you could wish to do with your network hardware is probably covered by systemd.link. The systemd.network configures the network (OSI layer 3). The systemd.netdev configures virtual networks; however, none of these unit files have the option to kick off a service after they're done, so I moved on.

### UDEV - unsuccessful

The udev filesystem has been around for almost 20 years now - it quickly took over for devfs in the early 2000s. When a device is plugged in that matches a udev rule, you can have it run a service by having this option in the line that matches your device:

```console
RUN="systemctl --no-block start <service>"
```

You may want to do this to load the new rule and get already plugged in devices run through that new rule:

```console
$ udevadm control --reload-rules
$ udevadm trigger
```

But this is moot. Network links aren't hardware - they're right above hardware in our OSI model. I don't remove my nic (or even adjust the power) when getting online or disconnecting. So a rule here would never fire. Next option.

### Tailing - successful

Instead, we can look at something happening in a file. If that thing shows up, we can do something and restart:

```console
$ systemctl cat test
# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test https://www.example.com

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

This sends `foobar` to syslogd to be processed, which writes to the syslog file we're monitoring. After it matches one and only one event, it stops running, and runs the ExecStopPost command. This echo shows up in journalctl (for our test but it could be any command). We can see this here:

```console
$ journalctl -xe -u test2 | grep echo
```

We can also have this event kick off another service as test has a Requires and After relationship to test:

```console
$ systemctl cat test*
# /etc/systemd/system/test2.service
[Unit]
Description=testing 2
Documentation=test2 https://www.example.com

[Service]
Type=simple
ExecStart=echo test2


# /etc/systemd/system/test.service
[Unit]
Description=testing
Documentation=test https://www.example.com
After=test2.service
Requires=test2.service
 
[Service]
Type=simple
Restart=on-success
ExecStart=sh -c 'tail -n1 -f /var/log/syslog 2>/dev/null | grep -q -m1 foobar'
```

But we need to enable, test, and list all the services we wish it to run in that file. So, we can reverse the dependency structure by putting RequiredBy in an [Install] section and enable the test2 service (instead of enabling the test service). However, to ensure the test service is running when enabling test2, we need to use BindsTo or Wants 

```console
$ systemctl cat test2
# /etc/systemd/system/test2.service
[Unit]
Description=testing 2
Documentation=test2 https://www.example.com
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

This setup allows us to not always directly manage test as test2 starts test for us. But test2 runs twice when it's triggered:

```console
$ systemctl cat test2
# /etc/systemd/system/test2.service
[Unit]
Description=testing 2
Documentation=test2 https://www.example.com

[Service]
Type=simple
ExecStart=echo test2

[Install]
RequiredBy=test.service
$ systemctl enable test2
Created symlink /etc/systemd/system/test.service.requires/test2.service → /etc/systemd/system/test2.service.
$ systemctl start test
```

So, we need to account for double events and manage the test service. See [4] for a logic diagram of Wants, PartOf, and Requires. There's a mapping of these associative properties and inverses here [5].

### Path - partially successful

Lastly, we can create a path file which monitors paths like:

```console
$ systemctl cat test.path
# /etc/systemd/system/test.path
[Unit]
Description=testing
Documentation=test https://www.example.com

[Path]
PathModified=/root/foo
Unit=test2.service

[Install]
WantedBy=multi-user.target
```

Which does as it's intended to and test2 kicks off when `/root/foo` is modified. However, when I try to do: `PathModified=/sys/devices/virtual/net/exttest0/operstate` nothing happens. The reason for this is because sysfs isn't an actual filesystem, it is just an interface to kernel memory. That isn't useful for what I'm trying to accomplish.

## Conclusion

It seems a bit overly complex to make an event-based service. It seems unnecessary to enable a service to install reverse dependencies. However, the service runner does a pretty good job for most cases. I do appreciate not having to write full wrapper scripts for my services. For this task, having a service that does:

```console
ExecStart=sh -c 'ip monitor link | grep ",UP,LOWER_UP"'
```

And letting that service kick off another service should do what I need well enough.

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

