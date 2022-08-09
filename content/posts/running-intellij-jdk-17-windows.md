---
title: "Running IntelliJ with JDK 17 on Windows"
date: 2022-02-01T23:12:12+01:00
draft: false
tags: ["intellij", "jetbrains", "phpstorm", "windows", "OpenJDK"]
---

# Make IntelliJ fast again

Running Jetbrains' IntelliJ IDEA with an alternative JDK can give you some performance boost. Based on the good writeup
for using
[JDK 17 with IntelliJ IDEA on MacOS with experimental `Metal` graphics support by Mustafa Akın](https://mustafaakin.dev/posts/2021-12-08-running-intellij-idea-with-jdk17-for-better-render-performance/)
I decided to give it a try on Windows, too.

You should have installed the newest version of IntelliJ IDEA / PHPStorm / Webstorm / etc. which is at the time of
writing 2021.3 or maybe even EAP / preview build.

## `Custom VM Options`

Go to `Help` -> `Edit Custom VM Options` and add the following call parameters:

```
--illegal-access=warn
--add-opens=java.desktop/java.awt.event=ALL-UNNAMED
--add-opens=java.desktop/sun.font=ALL-UNNAMED
--add-opens=java.desktop/java.awt=ALL-UNNAMED
--add-opens=java.desktop/sun.awt=ALL-UNNAMED
--add-opens=java.base/java.lang=ALL-UNNAMED
--add-opens=java.base/java.util=ALL-UNNAMED
--add-opens=java.desktop/javax.swing=ALL-UNNAMED
--add-opens=java.desktop/sun.swing=ALL-UNNAMED
--add-opens=java.desktop/javax.swing.plaf.basic=ALL-UNNAMED
--add-opens=java.desktop/java.awt.peer=ALL-UNNAMED
--add-opens=java.desktop/javax.swing.text.html=ALL-UNNAMED
--add-exports=java.desktop/sun.font=ALL-UNNAMED
--add-exports=java.desktop/sun.awt.image=ALL-UNNAMED
--add-exports=java.desktop/sun.java2d=ALL-UNNAMED
--add-exports=java.desktop/sun.awt.windows=ALL-UNNAMED
```

These will export all required java packages that are part of the proprietary Sun JDK and considered as private. JDK 17
still contains those but are not exported by default anymore. Trying to access them will result in an exception.

## Changing JDK version

Use the `Find Action` hotkey (`ctrl` + `shift` + `A` on Windows & Linux, `CMD` + `shift` + `A` on Mac) to search
for `Choose Boot Java Runtime for the IDE...`.

{{< figure src="/media/posts/intellij-switch-jdk17.png" alt="jdk-switcher dialog" >}}

In the opening dialog you can select the JDK you want to use. IntelliJ comes with several flavors of JDK 11, but you can
add any other compatible installed. IntelliJ should automatically pick up and suggest an installed JDK.
