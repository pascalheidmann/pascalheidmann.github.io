---
title: "Fixing not starting PHPStorm / other IntelliJ IDE"
date: 2021-08-07T22:34:56+02:00
draft: false
tags: ["intellij", "jetbrains", "phpstorm", "windows"]
---

## Problem

My PHPStorm does not start anymore, just sitting idle without any window
(windows 10)

## First try: running without 3rd party plugins

Locate your Intellij (or IntelliJ based IDE) executable. If you are using Jetbrain's toolbox go the bugged IDE'
s `Settings` page inside toolbox, expand `Configuration` and you should see something like

```shell
C:\Program Files (x86)\JetBrains\Toolbox\apps\PhpStorm\ch-0
```

Now open a terminal/windows cmd etc. and run

```shell
C:\Program Files (x86)\JetBrains\Toolbox\apps\PhpStorm\ch-0\phpstorm64.exe disableNonBundledPlugins
```

If you are lucky and your IDE starts disable all potentially offending 3rd party plugins and re-enable them one by one.

Sadly I wasn't so lucky this time...

## Solution

After checking the logs (`%HOMEDRIVE%%HOMEPATH%/AppData/Local/Jetbrains/PhpStorm2021.2/log/idea.log`) I found an error
at the end of the log file.

```log
java.net.BindException: Address already in use: bind
```

somewhere in Netty, a common java server framework which is used inside IntelliJ for internal communication.

As it turned out it tries to open a port between 6942 and 6991. Somehow no port was free anymore as there seems problem
with releasing ports with `winnat`
as [suggested in a docker bug report](https://github.com/docker/for-win/issues/3171#issuecomment-739740248).

If you encounter this problem you have to run following commands in an administrator elevated terminal

```shell
net stop winnat
net start winnat
```

or as an alternative

```shell
netsh winsock reset
```

if you don't mind restarting your computer.