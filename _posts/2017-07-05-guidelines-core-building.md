---
layout: post
title: Going crossplatform
subtitle: Guidelines to build good core code
---

# Learn another platform
I believe very strongly that you should know at least 2 platforms before architecting core code. Learning platform behaviour, issues and quirks teaches you what's expected on every platform. More importantly, by knowing multiple platforms you get a feeling on what can or should be shared.

# Don't reinvent the wheel
Do your research. :) There are lots of awesome libraries in the wild which will make your life a lot easier. In many cases these will already have gone through several stabilization cycles solving weird edge cases and unexpected behaviour you won't have to deal with anymore.

# Interface all the things
As a rule of thumb more code can be shared than expected. Business logic and services should be handled completely in shared code. Your future you will thank you if you can add new features, change the app flow or solve bugs once and get results on all platforms.

Expose your platform specific code through an interface, ideally combined into services with a clearly separated intent. Consider all platforms when devising return statements as not all clients will have the same result types or richness.

# Design for the least common denominators, allow for all possibilities

Occasionally platforms won't overlap in the exact same way. Consider these similarities and differences when writing your code. This comes into play for example for lifecycle events and navigation patterns. Ensure it's clear for all platforms what shared code is supposed to do or when it is supposed to be called.

# Consistency is key

This one is not necessarily cross platform related. Be consistent in everything: naming of properties, commands and events - structure of code across files - folder structure - ... Helps your sanity, development speed, maintainability and easiness of switching project scopes.
