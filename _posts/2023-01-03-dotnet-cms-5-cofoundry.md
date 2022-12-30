---
layout: post
title: Cofoundry
subtitle: A mini deep-dive in .NET (headless) CMS systems
---

This post is part of a bigger series. Read the [series introduction](/2023-01-03-dotnet-cms-1-intro/) for the full context.

I initially hesitated to add Cofoundry to the series since it's the odd one out in its approach. Where the other posts discuss CMS platforms which can be used out of the box, Cofoundry delivers the framework and the tools with which you can build everything yourself.

# Facts & figures

Cofoundry positions itself as follows:

> "Cofoundry is an unobtrusive ASP.NET Core CMS focused on code-first development and user-friendly content management. Run integrated, decoupled or headless, it's your choice."

Main differentiators:

- Compared to other CMS systems, Cofoundry provides you with the necessary framework and tools, but it requires you to put everything together in code.

Tech Stack:

- Open source / MIT licensed
- 750+ stars on GitHub
- Built with ASP.NET Core MVC / Angular pages
- (Azure) SQL Server as a backing database
- No API available out of the box

Cofoundry can only be self-hosted. It is intended to be added to your project through code.

# Headless CMS functionality

## Architecting data

Cofoundry is a code-first framework. Data schemas are created by creating some C# classes. Inside of these classes you can annotate your properties to build up the schema you want to work with. The 'title' property will be available by default, no need to add this manually in your class.

The data types you would expect are available out of the box through attributes Cofoundry provides. With known C# data annotations, you can further finetune expected behaviours.

```c#
  public class RecipeModel : ICustomEntityDataModel
  {    
      [MaxLength(500)]
      [MultiLineText]
      public string Description { get; set; }
      
      [Image]
      public int Image { get; set; }

      [Number(Step = "1")]
      [Range(1, 5)]
      public int Rating { get; set; }
  }

  public class RecipeCustomEntityDefinition : ICustomEntityDefinition<RecipeModel>
  {
      public string CustomEntityDefinitionCode => "RECIPE";
      public string Name => "Recipe";
      public string NamePlural => "Recipes";
      public string Description => "Recipes";
      public bool ForceUrlSlugUniqueness => true;
      public bool HasLocale => false;
      public bool AutoGenerateUrlSlug => true;
      public bool AutoPublish => false;
  }
```

## Creating data

From the content page you're able to create items for the available schemas. Once a draft is saved or the item is published you also get post metadata including an audit trail and version history.

![Create content](/img/dotnet-cms-cofoundry-3.png)

## Consuming data

There is no API generation out of the box because Cofoundry doesn't want to make assumptions on how you want to make your data available. There are however several utilities available in code to create your own controllers very quickly. On GitHub there is a full [headless sample](https://github.com/cofoundry-cms/Cofoundry.Samples.SPASite/tree/master/src/Cofoundry.Samples.SPASite/Api) which shows, for example, the built-in IApiResponseHelper and IContentRepository abstractions.

## User management / Authentication & authorization

For a headless CMS there are two types of authentication to look at: how do you manage your CMS users and how do you manage your data consumers.

Users are managed within the CMS itself. Roles can be created both through the admin user interface or through code. It is straightforward to set granular permissions on your roles from the user interface.

Since there is no default API created, the way you want to authorize your API is fully up to you. Cofoundry does give you all the tools to easily authorize your controllers on roles or permissions with a standard OAuth authorization flow.

# The developer story

## Getting started

There is a dotnet template available to get you quickstarted:

```bash
  $ dotnet new -i "Cofoundry.Templates::*"
  $ dotnet new cofoundry-web -n ExampleApp
```

Since Cofoundry only works with SQL Server you will have to run an instance locally to continue. The easiest to get started is to boot up a docker image locally and create a new database there.

```bash
  $ docker run -e "ACCEPT_EULA=1" -e "MSSQL_SA_PASSWORD=MyPass@word" -e "MSSQL_PID=Developer" -e "MSSQL_USER=SA" -p 1433:1433 -d --name=sql mcr.microsoft.com/azure-sql-edge

  $ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P MyPass@word
  1> CREATE DATABASE MyDB
  2> Go
```

After updating your connection string accordingly, the CMS will boot up. Out of the box you do get an example class called 'Product' which can be used as a reference for your own content types.

## Customization

As everything is done through code, by default everything you do is some form of customization. It should be possible to add your own content types, but this process is not documented currently so I wasn't able to try this out.

# Pros and cons of Cofoundry

Every system out there has benefits and downsides depending on your use case. Here are some items to take into consideration.

- The tools, utilities, and architecture Cofoundry gives you do work very well to quickly build a lot of functionality.
- Since everything is code-first, you can retain a full history of your data architecture through git.
- If you are purely looking for a headless CMS, Cofoundry is probably not for you. Cofoundry seems like a good choice if there are no other tools available with your exact requirements. It enables you to get a quick start to build upon.

<br />

# Resources

[https://www.cofoundry.org/](https://www.cofoundry.org/)
<br />
Official webpage of Cofoundry

[https://www.cofoundry.org/docs](https://www.cofoundry.org/docs)
<br />
Developer / user documentation for Cofoundry

[https://github.com/cofoundry-cms/cofoundry](https://github.com/cofoundry-cms/cofoundry)
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