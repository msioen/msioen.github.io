---
layout: post
title: Seamless QR scanning on Mac
subtitle: Introducing a new macOS utility application
---

So, I needed to scan some QR codes as part of my development flow. I'll be honest, I didn't do the homework. I'm sure there already are a lot of QR utilities out there. I did instead decide to roll out something of my own for several reasons:

- I wanted to be able to tweak the full experience to my needs.
- Reviewing new apps to find one which you like and trust could take a lot of time as well.
- It seemed small enough in scope to build myself and learn some things in the process.

Version 1.0 is a small status bar application with a very limited function set:

- You can select an area of your screen to scan for bar codes. 
- On success the parsed data is available on your clipboard / failure gives you a visual clue.
- It listens to <kbd>command</kbd> <kbd>shift</kbd> <kbd>B</kbd> to start scanning without using the menu.

I did get somewhat carried away releasing the first version without properly thinking about naming the application. This has as a result that the current name 'MenuBarCodeReader' is very dry and unoriginal. Suggestions for a proper name are welcome!

# Where can I find this

If you want to give this tool a go, you can find the application on [my GitHub](https://github.com/msioen/MenuBarCodeReader/releases). If you do try it out, please send some feedback my way. I've posted a screencast below showing how this version works.

![Demo](/img/demo.gif)

# Some technicals

The full application was built with Xamarin. While making it, I made two bindings of existing mac projects. These are also available for use on Nuget/GitHub.

- [ZXingObjC.Binding](https://github.com/msioen/ZXingObjC.Binding): binds the ZXingObjC library for barcode image processing.
- [HotkeyManager.Binding](https://github.com/msioen/JFHotkeyManager.Binding): hotkey manager, based on JFHotkeyManager

In a later post I'll go into a bit more detail about some parts of the development process.

# Into the future

Maybe somewhat differently than some of my other projects, I expect to use this quite a lot. This means I already put some thoughts in to where I want to go with this. If you have any other feature requests be sure to let them know.

- Auto update - looking at [Sparkle framework](https://sparkle-project.org) for this
- Preferences - customizable shortcut / send results to notifications / startup at boot / ...

<br />
<br />
