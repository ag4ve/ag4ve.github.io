---
pin: true
layout: post
title: "Ansible Debugging"
date: 2022-08-07
categories:                                         
  - code
tags:
  - code
  - ansible
  - python
image:
  path: /static/img/Western_Green_Lizard.jpg
---

I recently got into making a feature for an ansible module. Along the way, I obviously made bugs and needed to figure out how to find and fix those errors. This is everything from data nesting issues (obvious - print them and move them up or down) to causing issues trying to delete from the ansible parameters object/data structure. The bugs were pretty simple to fix, but not so simple to get a debugger for.

Ansible is python, so this basically means we’re talking about pdb. But how do I get that from ansible? The best ansible command line I found for testing is this:

> $ ANSIBLE_KEEP_REMOTE_FILES=1 ANSIBLE_DEBUG=True ansible-playbook -vvvv test.yml

Most of this just gives insane levels of output. There’s also the part to keep the artifact file ansible builds up around, but more about that in a bit. Most of this just gives insane levels of output, which is kinda what we want, right?

Well, kinda - but not really. what we really want is the ability to drop to an interactive debugger or at least log variables at certain places. It’s also possible the ansible logger module could work for this, but that’s more logic code than I wanted to bring in for a temporary function (I don’t want to keep much debugging in the code after I’m done). So how do we do this?

Well, the documentation (quite good) mentions putting the ANSIBLE_ARGS into a json file and running it like that.

```console
$ python ~/ansible_collections/community/test/plugins/modules/iptables.py \
  <<< '{"ANSIBLE_MODULE_ARGS": {"do":"save","chain":"INPUT","jump":"RETURN"}}'
```

Which works right up until you need to enter the debugger. And then nothing happens. I figure this is because the redirect for the input data file descriptor is grabbing something pdb uses, but we’re already trying to debug one thing, so I figure I shouldn’t go after two errors and keep it simple. Given this, lets do what the helpful documentation says to do here:

[https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html)

And stick our args into a file. We can then either run the module with:

> $ python -pdb ansible_module.py test_parameters.json

Which isn’t that useful unless you just want to step through each frame or something. Or, the much more useful way for me is to stick this line wherever you want to drop the debugger:

> import pdb; pd.set_trace()

But after you’re done with initial debugging of sample code, you go to create a play and run it and your module again blows up. And you look at all that data and think “how do I pass this to ansible to drop to a debugger now”? Since you still can’t run ansible and let it put you into a debugger.

My first thought was to use yq and jq to create the structure I wanted (don’t do this - this should basically work, but seriously don’t do this):

```console
$ yq -o=json test.yml \
  | jq --argfile file t.json '{ANSIBLE_MODULE_ARGS: .[0].tasks[0]."community.test.iptables"} \
  | .ANSIBLE_MODULE_ARGS.picker_definitions = $file' 
```

It requires knowledge of your role/plays, and knowledge of your module and only works for one play at a time. The better way to do this is to find the last ansible “remote” file you kept and grab the data from there and plop it in a file:

```console
$ grep ANSIBLE_MODULE_ARGS /home/azureuser/.ansible/tmp/ansible-tmp-1659789728.2394295-6951-276310893811636/AnsiballZ_iptables.py \
  | cut -d'=' -f2- \
  | sed -r "s/^ '//" \
  | sed -r "s/' *$//" \
  | jq > t3.json
```

I don’t think jq is required here, I just like it as a sanity check before creating a file that I want to use to debug something else. And then we can run our module with a normal python command and do the same run that we’d compiled a role for:

> $ python ~/ansible_collections/community/test/plugins/modules/iptables.py t2.json

