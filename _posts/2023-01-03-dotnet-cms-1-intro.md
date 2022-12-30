---
layout: post
title: A mini deep-dive in .NET (headless) CMS systems
subtitle: Series introduction
---

This is the first entry for a new blogpost series investigating headless Content Management Systems built on .NET. This series aims to give a good general overview of what the platforms have to offer. They will also contain relevant side notes or potential issues I ran into. They are not intended however to be a full, all-encompassing deep-dive or comparison.

The main focus will be on headless platforms, but several traverse the grey zone between full and headless. In some ways the backing technology of your headless CMS doesn't really matter since the focus is on data / API interaction. In this series however I'm focusing purely on platforms built on .NET. This limits the scope of the series somewhat but also enables familiar customization if you're working in the same technology stack.

# What is a CMS

Content Management Systems (CMS) have been around for a while. The main goal of a CMS is to create, manage, edit and publish content through a graphical user interface. Usually they are focused on website content where you able to create your website pages, templates and content without needing to know how to code. WordPress is probably the most widely used system currently.

There are many benefits to using a CMS as part of your solution. You're able to easily manage data even with limited technical knowledge. It's handy to allow multiple people access to the same data. It allows anyone with the rights to push new content to your website.

# What is a headless CMS

Historically most content management systems were built to straight-up support a website or even fully built the website from the CMS. The 'headless' concept decouples the end-product from the content. This means that a headless CMS purely focuses on the data itself and making it available through an API. It doesn't really care about how it's rendered or used down the line.

This approach has some benefits. It doesn't limit the use of your data to one consumer. It ensures you get systems that are more focused on data editing. It allows separate scaling when applicable.

A non-exhaustive list of some headless CMS options can be found here: [https://jamstack.org/headless-cms/](https://jamstack.org/headless-cms/).

# What do I look for in a headless CMS

Since a headless CMS decouples the data from how the data is used, the most important elements of a headless CMS are how the data is created and how it is consumed. These are the items I'm also going to focus on in the individual posts.

- data architecture: how can you define the data types, how many options are available, can you easily link data
- content creation: what does the content editing experience look like, what publishing options are available
- data consumption: how do we consume the data somewhere else, how are permissions and authorization managed

Aside from these you also have the general considerations you would have when investigating platforms. How stable is the system, what type of support can you rely on, do you want to self-host or use a SaaS solution, ...

# Does the technology matter?

Since a headless CMS is supposed to be decoupled from the consuming service or website the technology does not necessarily matter that much. The most important piece of the stack which does matter is how the data can be consumed and/or edited.

There are however many cases where some minor tweaks or additions in the CMS itself make a world of difference. In those cases, it can be handy to work with a system in the same technology stack you're used to working with.

<br />

---

Posts in this series:

- [Series introduction](/2023-01-03-dotnet-cms-1-intro/)
- [Squidex](/2023-01-03-dotnet-cms-2-squidex/)
- [Umbraco Heartcore](/2023-01-03-dotnet-cms-3-umbraco/)
- [Orchard Core](/2023-01-03-dotnet-cms-4-orchard/)
- [Cofoundry](/2023-01-03-dotnet-cms-5-cofoundry/)

<br />