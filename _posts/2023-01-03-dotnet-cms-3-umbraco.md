---
layout: post
title: Umbraco Heartcore
subtitle: A mini deep-dive in .NET (headless) CMS systems
---

This post is part of a bigger series. Read the [series introduction](/2023-01-03-dotnet-cms-1-intro/) for the full context.

# Facts & figures

Umbraco Heartcore positions itself as follows:

> "The flexible CMS that grows with your business"
> 
> "A headless CMS that's heartful and hardcore"

Main differentiators:

- The content management interface uses the main Umbraco CMS which is a battle-tested and widely used input interface
- There's a strong company backing and support ecosystem around Umbraco

Tech Stack:

- Full CMS: open source / MIT licensed - Headless CMS: closed source
- Full CMS: 3700+ stars on GitHub
- Built with ASP.NET Core MVC / Razor pages
- (Azure) SQL Server as a backing database
- You get GraphQL and REST options out of the box

Umbraco Heartcore is only available as a SaaS offering. You can self-host the full CMS, but any API functionality you would have to build yourself.

# Headless CMS functionality

## Architecting data

Data schemas are defined in the Settings tab. Umbraco offers quite a few different types here, but to keep things simple I'll focus on the 'Document Type' which covers the main functionality.

Inside of a Document Type you can add all the properties your data object should contain. Adding a name field is not necessary, every object will get this by default.

I did find that there are some protected property names like 'name'. Initially the system will allow you to create a field with these property names, but you will run into errors when trying to save.

The content type of your property is set by choosing an editor (for example Date/Time or Textarea). On an editor it's possible to set a configuration that defines more details on how the editor should behave. These configurations can be shared across properties. Validation rules are defined on the property itself.

![Create schema](/img/dotnet-cms-umbraco-1.png)
![Manage schema](/img/dotnet-cms-umbraco-2.png)

## Creating data

Something unique to Umbraco is that you must manage your content structure yourself where other headless systems usually automatically structure everything based on the content type. Coming from other systems I found this sometimes to be a bit counter-intuitive, but it does give you options to, for example, make separate logical groupings with the same content type children which could get different permissions and uses. This also means that during your architecture step you really should think about which schemas can be used as a root element and how nesting of content should happen.

Data is created in the 'Content' tab. Once you create a new data element you get the user interface you would expect. The editor as defined in your schema, a way to save data, publish data and some meta information. Umbraco also shows all the internal links to this content item and a full history.

![Create content](/img/dotnet-cms-umbraco-3.png)

## Consuming data

There is a small API Browser in the back-office which you could use to try out the content delivery and content management REST API. It's not really fleshed out however, I found using the API tooling I'm used to a lot handier. For actually consuming content I would seriously consider using GraphQL with Umbraco Heartcore. Everything which you need is available through REST but it's not always the nicest API to work with.

The GraphQL Playground which is offered gives a bit more features than the API Browser. You get the necessary documentation, schema definitions and autocompletion to build and test the queries you need.

![REST API](/img/dotnet-cms-umbraco-4.png)
![GraphQL API](/img/dotnet-cms-umbraco-5.png)

## User management / Authentication & authorization

For a headless CMS there are two types of authentication to look at: how do you manage your CMS users and how do you manage your data consumers.

Users are managed within the CMS itself. The best way to manage permissions is by creating groups and define the linked permissions here. Any user can then be added to one or multiple groups which will provide them the access they need.

Authorization for the API is directly linked to a user. Any user is able to create an API key which is used for any API communication.  Note that by default the API is accessible anonymously so be sure to protect it according to your expectations.

# The developer story

## Getting started

Umbraco Heartcore is only available as a SaaS offering. You can get a 14-day trial to validate how everything works. You can also get in touch with Umbraco itself to set up some calls to go over your needs and get relevant demonstrations of the technology.

If you want  to play around with the main CMS, you can do so locally. The [official documentation](https://docs.umbraco.com/umbraco-cms/fundamentals/setup/install) has all the information you need.

## Customization

The settings panel allows creating your own editors. This is called 'Grid Editors' in Umbraco Heartcore. You get the possibility to set the necessary JavaScript, HTML and CSS to build the layout you need. You can even tweak how your data is returned in the API by defining a JSON schema for your custom editor.

# Pros and cons of Umbraco Heartcore

Every system out there has benefits and downsides depending on your use case. Here are some items to take into consideration.

- The same content editing experience as the full CMS is used which is clean and works well.
- There is a huge community available to reach out to if you need any help.
- Umbraco has some very nice cloud features. The ability to have multiple environments for example is a very handy feature in production.
- When you start to use the system, you get a full end-to-end tour which will get you started with the basics.
- Since this is a SaaS offering you only have little customization possibilities, compared with the immense marketplace you get for the full CMS.
- To get the most out of Umbraco Heartcore you do need some Umbraco knowledge. There are some 'quirks' which you just must know for the best results.

<br />

# Resources

[https://umbraco.com/products/umbraco-heartcore/](https://umbraco.com/products/umbraco-heartcore/)
<br />
Official webpage of Umbraco Heartcore

[https://docs.umbraco.com/umbraco-heartcore/umbraco-heartcore](https://docs.umbraco.com/umbraco-heartcore/umbraco-heartcore)
<br />
Developer / user documentation for Umbraco Heartcore

[https://www.s1.umbraco.io/projects](https://www.s1.umbraco.io/projects)
<br />
Umbraco Cloud portal where you can manage your projects.

<br />

---

Posts in this series:

- [Series introduction](/2023-01-03-dotnet-cms-1-intro/)
- [Squidex](/2023-01-03-dotnet-cms-2-squidex/)
- [Umbraco Heartcore](/2023-01-03-dotnet-cms-3-umbraco/)
- [Orchard Core](/2023-01-03-dotnet-cms-4-orchard/)
- [Cofoundry](/2023-01-03-dotnet-cms-5-cofoundry/)

<br />