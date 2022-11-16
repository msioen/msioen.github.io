---
layout: page
title: Notes
subtitle: Terminal commands
---
## General

- take screenshot of active android device

```bash
  adb shell screencap -p | perl -pe 's/\x0D\x0A/\x0A/g' > screen.png
```

- parse json in vim / through python

```bash
  :%!python -m json.tool
```

- regex to delete x characters

```bash
  ^.{1,x}
```

- create password protected zip (will prompt for password)

```bash
  zip -er myfolder.zip myfolder
```

- export mov to gif

```bash
  ffmpeg -i test.mov -s 600x400 -pix_fmt rgb24 -r 10 -f gif - | gifsicle --optimize=3 --delay=3 > test.gif
```

<br />

## macOS

- setup dock properly

```bash
  defaults write com.apple.dock autohide-delay -int 0;killall Dock
```

- keep mac awake

```bash
  caffeinate -d
```

<br />

## Mqtt

- start mosquitto server (homebrew)

```bash
  launchctl load /usr/local/Cellar/mosquitto/1.4.14_2/homebrew.mxcl.mosquitto.plist 
  launchctl start homebrew.mxcl.mosquitto
```

- listen to all topics (also tests if mosquitto server was succesfully started)

```bash
  mosquitto_sub -v -t '#'
```

<br />

## DNS/Network/Certificates

- DNS checks

```bash
  dig A michielsioen.be
  dig CNAME www.michielsioen.be
```

- upload file to server

```bash
  scp myfile.txt root@127.0.0.1:/srv/www/files/files/myfile.txt
```

- verify ssl certificate

```bash
  openssl s_client -showcerts -connect michielsioen.be:443
```

<br />

## Git

- remove last N commits

```bash
  git reset --hard HEAD~N
```

- squash last N commits

```bash
  git rebase -i HEAD~N
```

- update last commit with currently staged changes

```bash
  git commit --amend
```

<br />
<br />

---

[Notes overview](../)
