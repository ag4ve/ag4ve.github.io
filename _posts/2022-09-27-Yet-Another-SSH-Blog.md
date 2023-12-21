---
pin: true
layout: post
title: "Yet Another SSH Blog"
date: 2022-09-27
author: shawn
categories:                                         
  - shell
tags:
  - security
  - ssh
  - os
image:
  path: /static/img/OpenSSH_logo.png
---

The purpose of this post is to give tips and tricks around the use of ssh and associated tools. I’m assuming you have used ssh, have run sshd (the server), and are somewhat familiar with a terminal. I’m not going to cover Windows much - PuTTY, MobaXterm, SecureCRT, and OpenSSH are generally available for Windows platforms but I don’t use them much so am going to generally ignore them.

This is also a broad topic and there are tons of ssh blogs so I don’t want to reinvent the wheel. Where possible, I’m just going to link to others’ writings and then I’ve got quite a few references at the end. There are subjects that I’m not going to cover right now as well: SSH CA (though some references talk about it), using a tun or tap, using openssl with ssh keys, paramiko.

## Keys

SSH relies on public key cryptography to establish connections. One may convert keys to and from X509, but X509 keys are not used in ssh key exchange. There are 4 key types to choose from: RSA, ED25519, ECDSA, DSA. Both a host and client key of the same type must exist for successful negotiation. Client keys are normally in $HOME/.ssh/id_{key type} and host keys are normally in /etc/ssh/ssh_host_{key type}_key. When you connect to a new server, by default your $HOME/.ssh/known_hosts file will be checked against the key the server offers and the connection will fail if the key doesn’t match what is in the file and you’ll be asked to confirm the server’s key fingerprint if it doesn’t exist. You may store the client key in an unencrypted file, password protect it with 128-bit AES, or read it from a smartcard. The server key must be stored as an unencrypted file (but a SSH CA may read from an HSM).

If a key is stored unencrypted, you are more likely to loose control of the key than have your ssh data compromised. That said there are a few things one should consider when determining what client and server keys your environment will support:

DSA has been depricated and shouldn’t be used for any reason. DSA was introduced because of US export controls with RSA that no longer exist and DSA is known to be unsecure. DSA also relies on 160 bits of entropy which may be exhausted with simple brute force (which could potentially compromise other services if allowed to exhaust entropy). https://github.com/brl/obfuscated-openssh/blob/master/WARNING.RNG

RSA is the tried and true solution. A minimum key length of 2k is recommended and 4k and 8k keys maybe desired and used. However, it takes much longer to negotiate connections with longer keys and is pretty annoying to work with an authorized_keys file with material from a single key taking up half of your screen. RSA is pretty universally supported in libraries and smartcard hardware though.

The later two types are ED25519 and ECDSA. ECDSA is older and ED25519 is faster preforming of the two. You may see ED25519 referred to as “curve 25519”. These keys are also much shorter than a decently long/secure RSA key and potentially still more secure.

There’s a short IETF draft with recommendations on what may be implemented. My only caviat is that you should probably know your clients as much as your servers when implementing ssh and so should only support the a minimum of the most secure schemas to support your users (and not everything “SHOULD” be supported): https://www.ietf.org/archive/id/draft-ietf-curdle-ssh-kex-sha2-14.html

Teleport (a paid SSH CA solution) has a decent explanation of ssh key types: https://goteleport.com/blog/comparing-ssh-keys/

But again, no matter the algorithm, if you have the ability to store keys in a secure enclave, you should probably prefer that vs more secure crypto algorithms.

## Verifying keys

The general way of verifying keys is with `ssh-keygen -lf file` (use the `-E md5` option to see the classic looking fingerprint). You may print the fingerprint of one key or many keys from a file.

For instance, if you want to see the fingerprints of all the keys your server may present to a client, you may do:
> for i in /etc/ssh/ssh_host_*_key; do ssh-keygen -lf "$i"; done

To see the fingerprint (plus some out of scope information) a ssh server has offered and that you’ve accepted when connecting to them, you can do:
```console
$ ssh-keygen -lf ~/.ssh/known_hosts
```
and look at the second column. Or simply do:
```console
$ ssh-keygen -lf ~/.ssh/known_hosts | cut -d’ ‘ -f2
```

If you’ve connected to a server from a client, one of those fingerprints should match. Under no circumstances should you disable fingerprint checking. If you do,then you should expect anomolies at the least and security events more likely. If you have hosts that frequently change, look into implementing an SSH CA server of which there are a number of free and paid solutions. Otherwise, you may have your build solution read the server key as demonstrated above and populate your own known_hosts file.

## Config files

SSH has two configuration files: /etc/ssh/sshd_config and $HOME/.ssh/config. The man pages for these are sshd_config and ssh_config respectively. I’ll first discuss the client config (because it’s more fun).

### Client config

Firstly, if you’d like to disable the use of you config file for one off commands for any reason you may do:  
```console
$ ssh -F /dev/null host
```

And things will work as you’d expect. However, the ssh config file is simple and pretty powerful. The file uses Host match options, which can contain leading and/or trailing wildcards and (most) options are first come, first serve. Which means, you can have a `Host *` at the bottom of the file to specify any options you’d like as default if you don’t otherwise set them. So lets look at some stanzas we may define:

```shell
Host *
  HostName 127.0.0.1
  Port 22
  User User
```
  
So, if I don’t match another Host with a HostName specified, I’m not going to try to connect to another host. I use this as a sanity check to ensure I know what I’m doing. I also don’t want to divulge my local username, so I connect as User@host unless I specify a username (another way to catch myself not thinking). But I do want to connect on the default ssh port (22) because I don’t want to sit waiting for the tcp connect failure message - I want immediate feedback when I mess up.

So then let’s set up the connection for github:

```shell
Host github.com
  HostName github.com
  User git
```
  
As long as that stanza comes before my catch-all `Host *`, I’ll be able to connect to github as normal:

```console
$ ssh github.com -T verify
Please provide the following verification token to GitHub Support.
ARAZ4WZ7M6M7W6FPEYYTBHTDHKQD5AVKMNZGKYLUMVSF6YLUZZRTCZN6VJYHKYTMNFRV623FPHHAIEONAU
```

Let’s then say that I have servers that I want to interact with normally but also want to do things like port forward. I don’t need to be verbose, so I can create most of the definitions in the initial stanza and then make other stanzas to do the port forward such as:

```shell
Host foo*
  HostName example.com
  User justme
Host foo-fwd
  LocalForward 55022 example2.com:22
```

And then, to establish the local forward, I just type:
```console
$ ssh -Nf foo-fwd
```

It sets up the forward, then returns me back to my prompt. If I want to ssh into my Android device’s linux chroot, I can do something similar (assuming the phone is connected to the same network):

```shell
Host termux*
  HostName 192.168.0.123
  Port 8022
Host *-proot
  RemoteCommand proot-distro login fedora -- /bin/bash -il
```

Then I’d type:
ssh termux-proot

And I’m presented with the fedora shell that’s on my phone.

Last, you should probably be using IdentitiesOnly and then specifying the IdentityFile you expect to be using when connecting to the server. However, if you use the ssh-agent, you’ll need to give it something to key off of. Do this with the following:

```console
$ ssh-add -L \
  | while read f; do \
    echo "$f" > ~/.ssh/scd-$(( i++ ))_rsa.pub; \
  done # writes the files w/ the pubkeys
```

You may also want to set timeout/keep alive and control master options in your config file.

### Server config

Most of the settings in your server sshd_config will probably be mandated by corporate policy or a security compliance. I’d recommend setting the `LogLevel DEBUG` and disabling all non-key auth (no PAM or passwords, etc).

I’d also recommend determining which key type you want to use in your environment, disabling and removing the other host keys, and regenerating the host key you wish to use with sufficiently secure criteria. This salt stack script gives some good indications on how to do this: https://github.com/cloud9ers/secure-sshd-salt/blob/master/secure-sshd.sls

There are two fun parts of the sshd config file: SSH CA related settings and AuthorizedKeysCommand. SSH CA is a larger topic that some of the posts I’ve referenced have covered and that I may further cover in the future. There’s also quite a bit of writing on PAM (privileged access management) as it relates to ssh that covers the CA system (which has nothing to do with the X509 CA system you may be familiar with). But outside of LDAP related topics, I haven’t seen AuthorizedKeysCommand covered elsewhere, so let’s talk about that. 

Every compliance framework I’ve seen talks about which crypto should be used and to enable a certain login `Banner` and how many keys to accept from a connection request and what to allow to be exported, etc. In other words, compliance policies mandate that controls are used for things that are hard to break, but they don’t have controls for key management, which is easy to mess up.

With the default `LogLevel` only a username is logged. If you want to see the fingerprint of keys that were attempted, you need to turn up the log level. Even if you log fingerprints, if you want to detect someone misusing their access, you’d need a mapping of fingerprints to “users” and I’ve never worked at a place that had this implemented.

Public keys are supposed to be public - they’re not passwords - it’s not against policy to share them. So is it ever against policy to add someone else’s key to my authorized_keys file? Even if it is against policy, we’ve already determined that there’s nothing that will detect this behavior.

Note: AIDE now has the ability to monitor a file within a wildcard list of directories with a configuration include like `/home/*/.ssh/authorized_keys` but none of the compliance frameworks I’m aware of mandate this. FIM is a totally separate topic, but this is a good primer on AIDE: https://www.malasuk.com/linux/advanced-intrusion-detection-environment-aide/

I’ve also written a post about writing an ssh worm, which assumes that either people leave keys laying around or that people forward ssh-agents to shared hosts. We can and should have a script/cron job to look for keys laying around by either matching the first line of the file or by matching on file type magic:

```console
$ file  ~/.ssh/id_rsa
/home/azureuser/.ssh/id_rsa: PEM RSA private key
$ cat ~/.ssh/id_rsa | head -1
-----BEGIN RSA PRIVATE KEY-----
```

However, how do we not allow people to forward ssh agents? There are four parts to a ssh public key:
> <options> <key type> <key material> <comments>

The key type is something like ssh-rsa or ssh-ed25519, etc. The key material starts with “AAAA” and is a base64 public key, and the comment is generally an email or hostname or username or some other human identifier. However, the field that goes before the key type is the most interesting piece: you can specify what a user, who logs in with that key, is allowed to do. So, let’s assume we want a user to only be able to login for normal shell access or rsync or scp or git, and not be able to do anything less common and less secure. That line in authorized_keys may look like:
> no-agent-forwarding,no-port-forwarding,no-user-rc,no-X11-forwarding ssh-rsa AAAAdeadbeef user@email.com

If we made each users’ authorized_keys file owned by root and not writeable by our user, that allows us to manage what the user can do and ensures there’s one key per user. We could even have our `AuthorizedKeysCommand` script check and log these permissions and option parameters, which is an improvement. But then there’s a file in the user’s home directory (`$HOME/.ssh/authorized_keys`) that may give a permission denied error when they try to manage their files (and that’s not nice). The nicer way would be to setup a user database in the `AuthorizedKeysCommand` script and return keys and parameters when the script is called with each user. We could even put logic to limit what times they can login. The script won’t be owned (or even accessible) by the user, so we know that things are being managed appropriately. There’s a decent indication of what you can do with this type of setup here: https://jpmens.net/2019/03/02/sshd-and-authorizedkeyscommand/

## The Command Line

From a command line, you’re generally going to want to write commands that reference your ssh_config. However, if you’re writing a script, it’s best to know what options are being used and not rely on a config file. It’s also good not to leave extra config files laying around. As such, I tend to define ssh options in a bash array and use array expansion to lay them all out for ssh commands like so:

```shell
ssh_opts=" \
  PasswordAuthentication=no \
  ServerAliveInterval=300 \
  ControlPath=~/.ssh/control-%r@%h:%p \
  ControlPersist=yes \
  ControlMaster=yes \
"
ssh -F /dev/null ${ssh_opts[@]/#/-o } host
```

It’s also annoying trying to figure out how to escape variables and handle line breaks for things that you want to run on a remote server. So I don’t. Instead write a function and then let your local shell define it for the remote shell and execute it there:

```shell
_f () {
  hostname
  time
  who
}
ssh host "$(declare -f _f) && _f"
```

It is often nice to be able to look at the differences between the same file on two different systems. You might not (probably shouldn’t) have access from one system to another, but can access both systems from a third host. One could copy files locally and diff them there, but that’s painful. Instead, a simple diff is nice, such as:
```console
$ diff -U0 <(ssh host1 cat /path/to/some/file) <(ssh host2 cat /path/to/some/file)
```

It’s also sometimes nice to look at log files on different hosts. While `ssh host tail -F /path/to/file` works well enough for a single host, redirecting lots of log files into one stream is generally unusable. Instead multitail will handle this nicely:
```console
$ multitail -s 2 -l 'ssh host1 "tail -f /path/to/log/file"' -l 'ssh host2 "tail -f /path/to/log/file"'
```

Sometimes keys aren’t managed well and you find servers you are supposed to manage, but are unable to log into. You may have a dozen private keys and there may be a number of user possibilities a server was setup with. The easy way to get into servers like this is with crowbar. While crowbar tries multiple keys, it doesn’t try those keys for multiple users, so we wrap it in a shell function:

```console
$ ucrowbar <host>
ucrowbar () {
  for user in azureuser ec2-user chef jenkins centos tomcat root ubuntu apache; do
    echo $user
    $HOME/gits/crowbar/crowbar.py \
      -b sshkey -s "$1" -u "$user" -k ~/.ssh \
      -o /dev/null -l /dev/null 2>&1 \
      | grep -vE \
        ' (Crowbar v|LOG-SSH:|START|STOP|No results found...)'
  done
}
```

## Resources

- I’ve written about having a linux VirtualBox environment on Windows when using a smartcard to authenticate ssh with: [https://ag4ve.blogspot.com/2019/04/linux-easily-native-desktop-environment.html](https://ag4ve.blogspot.com/2019/04/linux-easily-native-desktop-environment.html)

- I’ve written about creating an ssh worm with bash: [https://ag4ve.blogspot.com/2022/06/the-anatomy-of-bash-ssh-worm.html](https://ag4ve.blogspot.com/2022/06/the-anatomy-of-bash-ssh-worm.html)

- Mozilla has a great document on configuring a secure ssh server: [https://infosec.mozilla.org/guidelines/openssh](https://infosec.mozilla.org/guidelines/openssh)

- More ssh server configuration and background: [https://stribika.github.io/2015/01/04/secure-secure-shell.html](https://stribika.github.io/2015/01/04/secure-secure-shell.html)

- More ssh server configuration ideas:
  * [https://www.cyberciti.biz/tips/linux-unix-bsd-openssh-server-best-practices.html](https://www.cyberciti.biz/tips/linux-unix-bsd-openssh-server-best-practices.html)
  * [https://medium.com/@shashwat_singh/critical-controls-to-secure-openssh-installation-fab10fd43374](https://medium.com/@shashwat_singh/critical-controls-to-secure-openssh-installation-fab10fd43374)
  * [https://www.putorius.net/how-to-secure-ssh-daemon.html](https://www.putorius.net/how-to-secure-ssh-daemon.html)
  * [https://cryptsus.com/blog/how-to-secure-your-ssh-server-with-public-key-elliptic-curve-ed25519-crypto.html](https://cryptsus.com/blog/how-to-secure-your-ssh-server-with-public-key-elliptic-curve-ed25519-crypto.html)
  * [https://dl.dod.cyber.mil/wp-content/uploads/pki-pke/pdf/unclass-rg-open_ssh_public_key_authentication_20160217.pdf](https://dl.dod.cyber.mil/wp-content/uploads/pki-pke/pdf/unclass-rg-open_ssh_public_key_authentication_20160217.pdf)
  * [https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/infrastructure-services/OpenSSH/](https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/infrastructure-services/OpenSSH/)

- SSH traffic analysis: [https://www.trisul.org/blog/traffic-analysis-of-secure-shell-ssh/](https://www.trisul.org/blog/traffic-analysis-of-secure-shell-ssh/)

- I’ve written a host monitoring script that pretty much assumes ssh: [https://github.com/ag4ve/misc-scripts/blob/master/mon-hosts-packed](https://github.com/ag4ve/misc-scripts/blob/master/mon-hosts-packed)

- And a script that opens up a bunch of tmux sessions with ssh connections: [https://github.com/ag4ve/misc-scripts/blob/master/tmux-start.sh](https://github.com/ag4ve/misc-scripts/blob/master/tmux-start.sh)

- And a simple script to configure a remote terminal the same as your local one and start screen: [https://github.com/ag4ve/misc-scripts/blob/master/sshs.sh](https://github.com/ag4ve/misc-scripts/blob/master/sshs.sh)

- Technical background on keys:
  * [https://hyperelliptic.org/tanja/vortraege/Brazil_keynote.pdf](https://hyperelliptic.org/tanja/vortraege/Brazil_keynote.pdf)
  * [https://www.openssl.org/docs/man3.0/man1/openssl-speed.html](https://www.openssl.org/docs/man3.0/man1/openssl-speed.html)
  * [https://s3.amazonaws.com/files.douglas.stebila.ca/files/research/papers/AISC-MogSte16.pdf](https://s3.amazonaws.com/files.douglas.stebila.ca/files/research/papers/AISC-MogSte16.pdf)

