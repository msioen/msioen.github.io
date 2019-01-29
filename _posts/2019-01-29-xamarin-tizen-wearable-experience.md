---
layout: post
title: C# All The Things - Tizen Wearable
subtitle: 
---

This post isn't intended as a 'how to start developing for Tizen with Xamarin'. Samsung has some very good guides on this already which I linked at the bottom of this post.

Instead I'll list some general issues and tips. I'll also bundle easy access to various resources which were helpful to me when developing for Tizen Wearable.

# Setup

Xamarin development for Tizen wearables is enabled by Xamarin Forms. Getting started is rather fast. It only requires installing Tizen Studio and a Visual Studio extension: 'Visual Studio Tools for Tizen'.

As a reference, during development I was using Microsoft Visual Studio 2017 Version 15.6.1 with Visual Studio Tools for Tizen Version 2.3.0.0 on Windows. With this setup I was able to build/deploy/debug to emulator and device (with some caveats - see Debugging). 

According to the documentation you should be able to develop in Visual Studio Code as well. I was however unable to get VS Code integration working fully. Building the application and deploying worked fine but debugging wasn't at all possible and I didn't find how to set which certificates should be used when signing. In Visual Studio for Mac you're able to build Tizen apps but you can't deploy or debug as the extension doesn't exist here.

# Debugging

The debugging experience seems to be highly dependant on which device or emulator you're targeting. Locally I'm only able to get a full debug experience on a Tizen 5 emulator. When I debug on that device I'm able to put breakpoints where I want, step through code and inspect variables. Exceptions also result (mostly) in useful stacktraces.

Real devices and other emulators start in debug mode but the experience is not as rich. This seems to be due to some symbols which fail to load. In this case when something goes wrong you usually get a native segmentation fault exception without any hint as to where the exception comes from. Breakpoints, inspection of values and stepping through code work sometimes. If you need any of the failing symbols this stops working.

I also haven't been able to pipe debug/console output to any console or terminal window.

# User interface

UI-wise all the user interface controls from Xamarin Forms work as expected. On top of the default views, Samsung published some additional controls specifically targeting round devices and Samsung products with a bezel: Tizen.Wearable.CircularUI.

At time of writing you can't use Xamarin Forms prerelease versions (3.5 and higher) because the CircularUI library doesn't fully support it yet. They're actively working on this.

# Certificates

_Disclaimer: my development so far was exclusively for Galaxy Watch. Some info in this section could be different for other Tizen wearables._

Tizen applications need to be signed by a certificate to be able to install it. Unsigned applications can only be run on emulators. Creating a new certificate is done with the Certificate Manager ([Galaxy Watch documentation can be found here](https://developer.samsung.com/galaxy-watch/develop/getting-certificates/create)).

When creating a certificate it's important that the first screen you get is the choice to pick between a Tizen and a Samsung certificate. You have to choose Samsung here to get a functional certificate. Sometimes this option doesn't seem to show up. In those cases restarting can help. I've also had success with reinstalling the Samsung Certificate Extension if nothing else helped.

# Resources

[https://developer.tizen.org/development/guides/.net-application](https://developer.tizen.org/development/guides/.net-application)
<br/>
The official Samsung guides. There is a lot of very well documented information to be found here.

[https://github.com/Samsung/Tizen.CircularUI](https://github.com/Samsung/Tizen.CircularUI)
<br/>
[https://developer.tizen.org/development/guides/.net-application/wearable-circular-ui](https://developer.tizen.org/development/guides/.net-application/wearable-circular-ui)
<br/>
Samsung's open source Circular UI components. This is a collection of extra controls on top of Xamarin Forms specifically made to get the best UX on circular devices. There is also a sample project showing off all the views.

[https://github.com/Samsung/TizenFX](https://github.com/Samsung/TizenFX)
<br/>
C# bindings to native Tizen functionality. If you need specific functionality and you know the native api for it, it's easy to search in this library to find out if there's already a link available and what the matching C# function is.

[https://github.com/Samsung/Tizen-CSharp-Samples](https://github.com/Samsung/Tizen-CSharp-Samples)
<br/>
Samples maintained by Samsung showing a whole range of Tizen / Xamarin Forms functionalities in action.

[https://docs.microsoft.com/en-us/xamarin/xamarin-forms/platform/other/tizen](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/platform/other/tizen)
<br/>
[https://github.com/xamarin/Xamarin.Forms](https://github.com/xamarin/Xamarin.Forms)
<br/>
The official Xamarin documentation and source code.


<br />
<br />
