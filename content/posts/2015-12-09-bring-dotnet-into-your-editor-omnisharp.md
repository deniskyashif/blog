---
title: "Bring .NET Development into your favourite editor with Omnisharp"
date: 2015-12-09
draft: false
tags: ["csharp", "dotnet"]
summary: "IntelliSense as a service."
aliases: 
    - /2015/12/09/bring-.net-development-into-your-favourite-editor-with-omnisharp
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2015-12-09-bring-dotnet-into-your-editor-omnisharp.md"
---

This year the .NET ecosystem welcomed DNX (.NET Execution Environment). Now developers are able to compile to and execute MSIL code on any operating system. This led to the increase of the framework's popularity in places where Visual Studio isn't available. Also the emergence of Roslyn, the new .NET compiler platform, which exposes all the compile stages via an API, allowed powerful code analysis tools to be easily written. This unleashed the development of some amazing projects like Omnisharp. 

## What is Omnisharp?

Omnisharp is a series of tools and extensions, which include editor integrations and libraries, that together help developers work with C# in any editor under any operating system. It offers wide set of features like smart completion, refactoring, semantic highlighting and even shorthands for building and executing .NET applications. It was initially created back in 2011 by [Jason Imison](https://twitter.com/JasonImison), as a vim extension. However it kicked off big time in 2014, when Roslyn was released. The project is community driven and fully [open source](https://github.com/OmniSharp/omnisharp-roslyn), meanwhile there are some notable Microsoft folks involved, like David Fowler.

##  How does it work?

Omnisharp is basically a web server which exposes services via HTTP endpoints. It receives information about the workspace and its state, and has the intelligence to understand the code and return the appropriate information. There are two versions of the server. The initial one, [OmnisharpServer](https://github.com/OmniSharp/omnisharp-server), written in 2011, using NRefactory (C# static analysis library) and Nancy, and the new one [OmnisharpRoslyn](https://github.com/OmniSharp/omnisharp-roslyn), started in 2014, which utilizes Roslyn and uses WebAPI for its HTTP layer. OmnisharpServer tends to be a bit shaky and its development has stopped in favor of the Roslyn-based version. The latter also supports ASP.NET 5 (vNext) projects. 

![client-server](/images/posts/2015-12-09-omnisharp/client-server.png "Client-Server")

Then comes the client (editor plugin), which communicates with the server. Currently there are plugins for Emacs, Vim, Sublime Text, Atom and Brackets. Visual Studio Code comes with its built-in Omnisharp plugin. Integration instructions for each plugin are provided on the [project website](http://www.omnisharp.net/#integrations).

## Features

Below you can take a look at some of the features with Emacs as my editor of choice:  

* Smart completion  
<video alt="Smart completion" src="/images/posts/2015-12-09-omnisharp/intellisense.mp4" controls="controls" width="650px"></video>

* Find definition  
<video alt="Smart completion" src="/images/posts/2015-12-09-omnisharp/goto-definition.mp4" controls="controls" width="650px"></video>

* Find implementation  
<video alt="Smart completion" src="/images/posts/2015-12-09-omnisharp/goto-implementation.mp4" controls="controls" width="650px"></video>

* Fix usings  
<video alt="Smart completion" src="/images/posts/2015-12-09-omnisharp/fix-usings.mp4" controls="controls" width="650px"></video>

* Format code  
<video alt="Smart completion" src="/images/posts/2015-12-09-omnisharp/code-formatting.mp4" controls="controls" width="650px"></video>

## Looking forward

Omnisharp is still a young project and currently works only with C#. Even though it started in 2011, it hasn't progressed much until a year ago, so there is a lot more to come. This June at the NDC Conference in Oslo Mathew McLoughlin, an Omnisharp team member, said that they continue to add new features to the server in terms of refactoring and code navigation. With the ability to pull in more libraries, the refactoring support is going to get even better. He also announced that the team works on adding [scriptcs](http://scriptcs.net/) support and more editor extensions. Apart from Razor view support, probably the biggest feature still missing is the debugging, but the project is being actively developed and the community is constantly growing, so we should see them coming in some time as well. 
