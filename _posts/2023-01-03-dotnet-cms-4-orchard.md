---
layout: post
title: Orchard Core
subtitle: A mini deep-dive in .NET (headless) CMS systems
---

This post is part of a bigger series. Read the [series introduction](/2023-01-03-dotnet-cms-1-intro/) for the full context.

# Facts & figures

Orchard Core positions itself as follows:

> "Open-source, modular, multi-tenant application framework and CMS for ASP.NET Core"

Main differentiators:

- Orchard Core is not just a headless CMS, it's a great framework for modular applications
- Focused on modularity and own customizations
- Very nice GraphQL experience

Tech Stack:

- Open source / BSD 3-Clause licensed
- 6300+ stars on GitHub
- Built with ASP.NET Core MVC / Razor pages
- Supports SQL Server, MySQL, PostgreSQL and SQLite (through a document abstraction layer called YesSql)
- GraphQL API out of the box

Orchard Core as a CMS can only be self-hosted. It is intended to be added to your project through code.

# Headless CMS functionality

## Architecting data

The first time you start creating a new Content Type in Orchard you may be somewhat overwhelmed as there are a lot of options. In essence however this works similar to other systems where you define all the content fields you expect your schema to have. I do have the feeling that the default field types are missing however missing some options which competing systems would have out of the box like more in-depth validation options. But the items you're missing can be added by customizing content types for yourself.

Aside from the general content schemas with their own fields, Orchard Core invests heavily in more reusable concepts through something called 'content parts'. This allows you to easily add the same behaviour and properties so multiple items. Several content parts are predefined to get you started going from a simple title item to more complex list behaviours.

![Create schema](/img/dotnet-cms-orchard-1.png)
![Manage schema](/img/dotnet-cms-orchard-2.png)

## Creating data

There are no surprises in the actual content editing experience, everything works the way you would expect. I do like the way Orchard Core deals with drafts and published versions, that's setup in a clean and easy to understand way.

![Create content](/img/dotnet-cms-orchard-3.png)

## Consuming data

Data from Orchard Core is consumed through the built-in GraphQL integration. The GraphQL playground is top-notch, even allowing you to build up your queries by just clicking through on the objects.

![GraphQL API](/img/dotnet-cms-orchard-5.png)

## User management / Authentication & authorization

For a headless CMS there are two types of authentication to look at: how do you manage your CMS users and how do you manage your data consumers.

As with the other CMS systems the main setup happens in the CMS itself by defining roles, permissions and users directly. It's however also possible to connect an external user directory through OpenID Connect while providing your own mapping to the CMS defined roles. On a role level in the settings panel you can set very fine-grained rights as to what users are allowed to do.

Calling the GraphQL endpoint externally requires you to authenticate through a standard OAuth flow. The user should have a role with the ExecuteGraphQL permission.

# The developer story

## Getting started

Creating a new CMS with Orchard is extremely easy. From a clean project you just need to reference a nuget reference and add two lines to the Program.cs file.

```bash
  $ dotnet new webapp
  $ dotnet add package OrchardCore.Application.Cms.Targets --version 1.5.0
```

```c#
  var builder = WebApplication.CreateBuilder(args);

  builder.Services.AddOrchardCms();

  var app = builder.Build();

  app.UseStaticFiles();
  app.UseOrchardCore();

  app.Run();
```

```bash
  $ dotnet build
  $ dotnet run
```

When you navigate to your localhost url the first time you'll get a setup page. To use Orchard Core as a headless CMS you can choose the 'headless site' recipe here which will enable all the features you need to be successful.

![Orchard Core Setup](/img/dotnet-cms-orchard-6.png)

## Customization

If you zoom in on the CMS aspects the option to create your own content types or create custom editors for existing content types is [built-in](https://docs.orchardcore.net/en/latest/docs/reference/modules/ContentFields/#creating-custom-fields). 

Aside from these types of customizations, Orchard Core is a modular platform built for extensibility in general.

# Pros and cons of Orchard Core

Every system out there has benefits and downsides depending on your use case. Here are some items to take into consideration.

- Adding Orchard Core to your .NET project is extremely easy.
- It's actively being developed with a very transparent roadmap and visibility.
- To use Orchard Core as a choice for CMS, it seems to be missing some components and options I would expect out of the box. It's for example not possible to limit a media picker to only accept images. To get the best experience you will have to make some customizations. The core framework itself is a great enabler however.
- There is lots of documentation but some of it can be very technical or hard to immediately apply.

<br />

# Resources

[https://orchardcore.net/](https://orchardcore.net/)
<br />
Official webpage of Orchard Core

[https://docs.orchardcore.net/en/latest/](https://docs.orchardcore.net/en/latest/)
<br />
Developer / user documentation for Orchard Core

[https://github.com/OrchardCMS/OrchardCore](https://github.com/OrchardCMS/OrchardCore)
<br />
Source code


<br />

---

Posts in this series:

- [Series introduction](/2023-01-03-dotnet-cms-1-intro/)
- [Squidex](/2023-01-03-dotnet-cms-2-squidex/)
- [Umbraco Heartcore](/2023-01-03-dotnet-cms-3-umbraco/)
- [Orchard Core](/2023-01-03-dotnet-cms-4-orchard/)
- [Cofoundry](/2023-01-03-dotnet-cms-5-cofoundry/)

<br />