---
title: "Debug Java via VSCode"
date: 2020-02-15T10:19:20+08:00
modified: 2020/02/15
tags: ['Debug', 'Java']
keywords: ['Java', 'Debug']
category: ['Technology']
---

## Background

Since last year I moved to vscode from emacs\(yeah u r right, I betray my belief cause vscode have emacs keymap.\) for some big project written in python, java and C/C++. The reason why I didn't use IDEA is because it's just a new f**king resource monster just like myeclipse in year 2009\(I have to say that without SSD IDEA is a bullshit.\).

With the remote server plugin, it's very convenient to develop and debug projects that host on remote server.
It works very well for python or C/C++ project cause usually you can easily start your project in debug mode from the entry point. But java, things are different cause java project usually have many different build system like maven, ant, gradle. For me I'm not very familiar with them, and the reason I begin to write java is to enhance a static java analysis project based on flowdroid.

Now the problem comes, flowdroid use maven to build, so usually it's very hard for me to directly debug it via press **F5**. So I searched a lot in Google and have solution finally.

## Debug remote java project via attach

**F**irst make sure you've already build your project successfully in to a jar file with all dependencies. If you build a jar without the dependencies, you may have to specify **classpath** to let Java can find them.

Then here is the way to start them in debug mode.

```bash
java -agentlib:jdwp=transport=dt_socket\
,server=y,suspend=y,address=127.0.0.1:6666 -jar [YOUR JAR FILE]
```

If the jar file have all dependencies satisfied. It will be launch but wait for debugger to attach.

Then in your vscode workspace, add below entry to launch.json in your workspace's .vscode folder. If there is no launch.json, create it.

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "java",
            "name": "Debug (Attach) - Remote",
            "request": "attach",
            "hostName": "127.0.0.1",
            "port": 6666
        }
}
```

Now feel free to debug your project with any breakpoints on your source files. :)
