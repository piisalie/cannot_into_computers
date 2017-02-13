---
title: Command Line History Hacking
date: 2017-02-13 22:17 UTC
tags: Shell, Bash, Command Line, Linux, other
---

Lately I've been interested in optimizing my use of the one true shell: Bash.

One nifty feature I learned from the wonderful ["From Bash to Z Shell"](http://www.apress.com/us/book/9781590593769) is how to use the command line history for command composition.

In addition to `C+p`, `C+n`, `C+r` and the like for navigating and reusing past commands, you can also compose new commands from past commands using the "bang" history lookup modifiers.



## The bang lookup

`$ !!` is super simple, and will return the last command. You can use this in combination with `$ history` to find and invoke a specific line from the your command history.

`$ history 10` will return the last 10 lines of history, with their line identifiers. If you have a line like `1993  cd sites/hanabi/the_neverending_application/` you can then re execute that command by using the bang plus line number: `$ !1993`. But WAIT, there's more:
You can also select certain words from past commands.



## Command word/argument look up

It is also possible to reuse only certain parts of the last command:

`!:0` will return the first word of the last command (usually the invoked command), observe:

```shell
$ ls ~
$ echo !:0
```
This will invoke `echo ls`.

You may also use other numbers to reuse other parts of the command: `!:1` would return `~`. These are 0 indexed because computers.

You can also use the `!:*` to get just the arguments of the last command



## N-1 History
`!-2:0` will go two lines into the command history instead of to just the last one. Similarly you can do `!-3:0` or `!-4:0`. This is helpful if you're working on building up a command, and want to keep referencing the same line.



## Find and Replace
The best thing you can do with command history is replace bits of the command you're reusing. Let's say you just spent the last 20 mins building up this supremely perfect copy command:

```shell
$ cp /lol/some/super/nested/directory/file.txt /var/log/other/place/that/is/probably/annoying/file.txt
```

Then after running it remember you had wanted to rename the file too. Not to worry, you can do find and replace on reused portions of commands.

```shell
$ mv !:2 !:2:s/file.txt/what_i_actually_wanted.txt/
mv /var/log/other/place/that/is/probably/annoying/file.txt /var/log/other/place/that/is/probably/annoying/what_i_actually_wanted.txt
```

Disclaimer: you should probably not use these in combination with things like `rm -rf` or other activities that are difficult to undo. Good luck and happy command line history modifying.
