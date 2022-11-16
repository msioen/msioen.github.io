---
layout: page
title: Cheatsheet
subtitle: Terminal commands
---
## General

- take screenshot of active android device

```
adb shell screencap -p | perl -pe 's/\x0D\x0A/\x0A/g' > screen.png
```

- parse json in vim / through python

```
:%!python -m json.tool
```

- regex to delete x characters

```
^.{1,x}
```

- create password protected zip (will prompt for password)

```
zip -er myfolder.zip myfolder
```

- export mov to gif

```
ffmpeg -i test.mov -s 600x400 -pix_fmt rgb24 -r 10 -f gif - | gifsicle --optimize=3 --delay=3 > test.gif
```

<br />

## macOS

- setup dock properly

```
defaults write com.apple.dock autohide-delay -int 0;killall Dock
```

- keep mac awake

```
caffeinate -d
```

<br />

## Mqtt

- start mosquitto server (homebrew)

```
launchctl load /usr/local/Cellar/mosquitto/1.4.14_2/homebrew.mxcl.mosquitto.plist 
launchctl start homebrew.mxcl.mosquitto
```

- listen to all topics (also tests if mosquitto server was succesfully started)

```
mosquitto_sub -v -t '#'
```

<br />

## DNS/Network/Certificates

- DNS checks

```
dig A michielsioen.be
dig CNAME www.michielsioen.be
```

- upload file to server

```
scp myfile.txt root@127.0.0.1:/srv/www/files/files/myfile.txt
```

- verify ssl certificate

```
openssl s_client -showcerts -connect michielsioen.be:443
```

<br />

## Git

- remove last N commits

```
git reset --hard HEAD~N
```

- squash last N commits

```
git rebase -i HEAD~N
```

<br />
<br />

---

[Cheatsheet overview](../)
