---
pin: true
layout: post
title: "Bash Patterns"                                       
date: 2017-04-16
categories:                                         
  - bash
  - script
tags:
  - code
  - script
image:
  path: /static/img/The.Matrix.glmatrix.2.png
---

### Audience

I assume some prior Unix command line experience

### Note

I've done much more work to the slide deck that derrived from this post and have found some errors in this post along the way that I haven't gotten around to fixing. Until I do, I'd recommend referring to my [presentation slides](https://ag4ve.github.io/bash-patterns/#/).

I keep running into "bash scripts" that look like old style shell scripts - capitalized variable names, tons of subshells piping all over the place, etc. This does not need to be the case - bash scripts can be written with some elegance.

# Bash Patterns

There are some design patterns that I think may help in this effort (and may also help scripts run faster).

## Help

First, in bash, most things have a help document:

```console
[swilson@localhost ~]$ help help | head -4
help: help [-dms] [pattern ...]
    Display information about builtin commands.
 
    Displays brief summaries of builtin commands.  If PATTERN is
```
{: file="help help" }

```console
[swilson@localhost ~]$ help [ | head -4
[: [ arg... ]
    Evaluate conditional expression.
   
    This is a synonym for the "test" builtin, but the last argument must
```
{: file="test help" }

```console
[swilson@localhost ~]$ help [[ | head -4
[[ ... ]]: [[ expression ]]
    Execute conditional command.
   
    Returns a status of 0 or 1 depending on the evaluation of the conditional
```
{: file="new test help" }

```console
swilson@localhost ~]$ help declare | head -4
declare: declare [-aAfFgilnrtux] [-p] [name[=value] ...]
    Set variable values and attributes.
   
    Declare variables and give them attributes.  If no NAMEs are given,
```
{: file="declare help" }

```console
[swilson@localhost ~]$ help set | head -4
set: set [-abefhkmnptuvxBCHP] [-o option-name] [--] [arg ...]
    Set or unset values of shell options and positional parameters.
   
    Change the value of shell attributes and positional parameters, or
```
{: file="set help" }

```console
[swilson@localhost ~]$ help enable | head -4
enable: enable [-a] [-dnps] [-f filename] [name ...]
    Enable and disable shell builtins.
   
    Enables and disables builtin shell commands.  Disabling allows you to
```
{: file="enable help" }

These documents are generally about a page long and do not go through a pager so may scroll.

Note: [ is also an executable command on most Unix systems - bash will use the internal [, but it's good to realize that it's there:

```console
[swilson@localhost temp]$ whereis [
[: /usr/bin/[ /usr/share/man/man1/[.1.gz
```
{: file="whereis test command" }

## Variable expansion and itteration

I so often see things like:
```
for i in $(seq 0 5); do ...; done
```

in scripts. Not only is creating a subshell costly, but it's extra typing, extra reading, and just looks ugly. Bash will expand variables for you:
```
echo {0..5}
```

or:
```
for i in {0..5}; do ...; done
```

But it gets better. Say you notice a host computer3.example.com and you wonder if that means there are also computer1.example.com, computer2.example.com, etc. This is trivial to check:
```console
$ dig computer{1..5}.example.com
```

As the whole word is expanded out with each itteration. Such as:

```console
[swilson@localhost ~]$ echo foo{0..5}
foo0 foo1 foo2 foo3 foo4 foo5
```
{: file="word expansion" }

However, maybe you're saying that seq allows you to itterate over only even numbers and that's why you prefer it and that I can't do that here? Well you'd be almost right:

```console
[swilson@localhost ~]$ for ((i = 0; i <= 10; i+=2)); do echo -n "$i "; done; echo
0 2 4 6 8 10
```
{: file="c style for" }

## Math

That last example brings up a point - modern bash can handle anything that the expr command can:

```console
[swilson@localhost ~]$ echo $((5 % 2))
1
[swilson@localhost ~]$ echo $((5 / 2))
2
[swilson@localhost ~]$ echo $((5**2))
25
```
{: file="math" }

And then some:

```console
[swilson@localhost ~]$ echo $(( (5**2) < 30 ))
1
```
{: file="more math" }

## Command test

Determining if a command is installed:

When writing a script meant for someone else to run, it's generally smart to determine if a command is installed. To do this, you want something like this:

```console
if ! command -v printf >/dev/null 2>&1; then
  echo "We can not find the bash printf command" >&2
  exit 1
fi
```
{: file="command exists" }

Eventually you may even want to create a "die" function, as typing that again and again can get a bit repetative (see mon-hosts linked at the bottom for an example of what this function may look like).

## Reading files as input

If you are writing a script and want to have a config file, just source another file and use those variables like:
```
source ~/.config/file >/dev/null 2>&1
```

I only have to redirect STDERR so that I don't show an error message if the sourced file doesn't exist (I don't need to check for it first if I do this), but just in case someone does something weird in the config file, might as well redirect STDOUT too.

Note: calling it a config file makes no difference to bash - it executes it just like any other script. This means, you should treat any config file you source as executable. In other words, if you put this in a config file you source:
```
FOO=$(rm -rf /*)
```

You will loose data and store the results of your lost system in $FOO. Or to demo this:

```console
[swilson@localhost temp]$ echo "echo foo" > sourced
[swilson@localhost temp]$ source ./sourced
foo
```

### CSV Manipulation

Within a script, if you need other file data, you can do this:

```console
[swilson@localhost temp]$ echo $(< foo.csv)
foo,bar,baz aaa,bbb,ccc
[swilson@localhost temp]$ echo "$(< foo.csv)"
foo,bar,baz
aaa,bbb,ccc
[swilson@localhost temp]$ mapfile -t -c 1 -C 'echo $@' < foo.csv
0 foo,bar,baz
1 aaa,bbb,ccc
```
{: file="reading csv" }

The first two can be used to define a variable, and the later can be used to run each line through a function, which is the same as:

```console
[swilson@localhost temp]$ while read line; do echo $line; done < foo.csv
foo,bar,baz
aaa,bbb,ccc
```
{: file="reading a csv file" }

Though mapfile has options to start from a line, how many lines to read per function call, etc which while doesn't really provide.

We can also easily manipulate a *simple* csv file with either while or mapfile (I'll demonstrate while):

```console
[swilson@localhost temp]$ while IFS="," read -a data; do echo "${data[0]}"; done < foo.csv
foo
aaa
'''
{: file="manipulate csv" }

Or directly from a string:

```console
[swilson@localhost temp]$ while IFS="," read -a data; do echo "${data[0]}"; done <<< "aaa,bbb,ccc"
aaa
```
{: file="manipulate csv string"}

## Sane defaults

After you're done looking through a config file (but before you take command line options), it's generally smart to define sane defaults. Most people do this with:

> [[ -z foo ]] && foo="something"

But this is way too much typing and not very elegant. Instead, try:
```
: "${foo:="something"}"
```

This is similar to the `||=` or `//=` operator from other languages

Defining non-printable characters

Normally, I only need to define a newline character to be inserted in the middle of a string, such as:
```
nl=$'\n'
```

However, this same method can be used to define/print whatever character you want:

```console
[swilson@localhost temp]$ for i in {0..255}; do eval "echo -n $'\x$i'"; done; echo
         !"#$%&'()0123456789@ABCDEFGHIPQRSTUVWXY`abcdefghipqrstuvwxy0123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789 0 1 2 3 4 5 6 7 8 9!0!1!2!3!4!5!6!7!8!9"0"1"2"3"4"5"6"7"8"9#0#1#2#3#4#5#6#7#8#9$0$1$2$3$4$5$6$7$8$9%0%1%2%3%4%5
```
{: file="fuzzing" }

## Variable manipulation and commands

Since version 3 (IIRC) bash has had support for basic arrays. These are quite simple to use and pretty powerful. Either
```
declare -a foo
```

or just:
```
foo=(aaa bbb ccc)
```

You can also manipulate arrays with globs, such as:

```console
[swilson@localhost temp]$ echo "${foo[@]/a/A}"
Aaa bbb ccc
[swilson@localhost temp]$ echo "${foo[@]/a*/A}"
A bbb ccc
[swilson@localhost temp]$ echo "${foo[@]/#a/A}"
Aaa bbb ccc
```
{: file="alter an array" }

Or evaluate the array as a string and manipulate the whole thing:

```console
[swilson@localhost temp]$ foo=(l h)
[swilson@localhost temp]$ ls ${foo/#/-}
total 8
-rw-rw-r--. 1 swilson swilson  24 Apr 10 18:56 foo.csv
-rwxrwxr-x. 1 swilson swilson 601 Apr 10 19:45 t.sh
```
{: file="array in a command" }

It should be apparent at this point that any array may be passed as a parameter to any program, or even be used as a whole to execute something. It should also be noted that, because of this, any unquoted variable should probably be treated with the same care as an eval statement. (Caveat: variables in [[...]] and ((...}} are treated like a variable and are not used to run commands)

You can also easily check whether an element exists in an array by using the same general princible of evaluating it as a string:

```console
[swilson@localhost temp]$ foo=(aaa bbb ccc)
[swilson@localhost temp]$ [[ " ${foo[@]} " =~ " bbb " ]] && echo ok
ok
[swilson@localhost temp]$ [[ " ${foo[@]} " =~ " eee " ]] && echo ok
[swilson@localhost temp]$
```
{: file="string in array" }

## Functions

Functions are awesome (and present in bash) and not utilized as much as they should be. While an alias can be used to expand what a word can do in bash:

> alias less='less -R' # display ANSI color

A function has lot more power

```shell
# long dig
ldig () {
  dig +trace +nocmd "$@" any +multiline +answer
}
```
{: file="long dig function" }

As parameters go in the middle of the parameters instead of to one side or the other.

It's also worth noting that you can't get around using an alias on the command line, whereas if you prepend a function command with a '\', bash executes the actual command. This makes working with half baked ideas much nicer :)

Or lets say I want to run some fancy logic as sudo or with ssh on a remote machine. Handling quoting here can be quite the pain. It also looks really dirty. There is, however an alternative:

```console
[swilson@localhost temp]$ _f () { whoami; }
[swilson@localhost temp]$ sudo bash -c "$(declare -f _f) && _f"
root
[swilson@localhost temp]$ _f () { groups $1; }
[swilson@localhost temp]$ sudo bash -c "$(declare -f _f) && _f tuser"
tuser : tuser
```
{: file="remote function" }

I can also pass in variables and the like using the same mechanism. The following comes from a consul-template script and shows passing a function along with a variable and writing output from that to a file.

```shell
echo declare -A saveidx="'("$( \
  {{range tree (printf "/%s/%s" $env $prefix)}} \
    consul lock "$lockpath/{{.Key}}" "$(declare -f _autobs) && $(declare -p saveidx) && _autobs \"{{.Key}}\" \"{{.Value}}\" \"{{.ModifyIndex}}\"" & \
  {{end}}\
)")'" > "$store"
```
{: file="consul template snip" }

You can also pass arrays to functions:

```console
[swilson@localhost temp]$ func () { echo "zero [${!0}] one [${!1}] two [${!2}] three [${!3}]"; }
[swilson@localhost temp]$ func foo[@] i
zero [] one [aaa bbb ccc] two [5] three []
```
{: file="variable reference" }

### Data from functions

As mentioned earlier, subshells are expensive (so try not to use them). In other words, don't do this:

```shell
func () {
  echo "something"
}
foo=$(func)
```
{: file="subshell" }

Instead, because defining variables is pretty cheap, define global return variables for functions, either with:

```shell
declare F_RET=""
f () {
  F_RET=""

  ......
}
```
{: file="return variable" }

or with newer bash, you can just do:

```shell
f () {
  declare -g F_RET=""
}
```
{: file="new return variable" }

This also means you can return an array if you need to and do not need to worry about how to recompile a returned string into an array. As a side node, newer bash also allows you to declare an associative array (also known as a hash or dictionary or lookup table) with declare -A.

### Redefine functions

Functions can also be redefined on the fly, so:

```shell
f () {
  f {} {
    echo "foo"
  }
}
```
{: file="replace a function" }

Will only print something the second time you run the function. Though you could also always just call f at the end of the outer function.

## Debugging

In a script, setting the 'e' and 'x' flag will show verbose output - this can also be set and unset at the beginning and end of a function, such as:

```shell
f () {

  set -ex

  ....

  set +ex

}
```
{: file="set unset in function" }

It is also good to realize that the same declare statement used to pass in variables and functions to a subshell may be used to show you what bash thinks the function or variable looks like as well:

```console
[swilson@localhost temp]$ i=(aaa bbb ccc)
[swilson@localhost temp]$ declare -p i
declare -a i='([0]="aaa" [1]="bbb" [2]="ccc")'
[swilson@localhost temp]$ f () { echo; }
[swilson@localhost temp]$ declare -f f
f ()
{
    echo
}
```
{: file="read declare" }

## Bonus

Handling programs that want to read from a file:

> echo "foo" | cat /dev/fd/0

This also works over ssh and sudo (as long as you don't redirect STDIN or anything).

So for instance, with ssh, I have a nice little alias that disables everything in my config file:

> alias sshn="ssh -F /dev/null"
> alias scpn="scp -F /dev/null"

Published scripts

A while ago I published two scripts that should highlight most of what I mentioned in here

* [mon-hosts](https://github.com/ag4ve/misc-scripts/blob/master/mon-hosts-packed)
* [tmux-start](https://github.com/ag4ve/misc-scripts/blob/master/tmux-start.sh)

