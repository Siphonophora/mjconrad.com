---
layout: single
title: My Stack
permalink: /my-stack/
---

This is an occasionally updated list of tools and libraries I use. I'm skipping a bunch of the most common tools for brevity.

### Visual Studio Extensions:

* [Add New File](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.AddNewFile64) Simple file creation, I rely on this so much VS feels broken without it.
* [Show Inline Errors](https://marketplace.visualstudio.com/items?itemName=AlexanderGayko.ShowInlineErrors) Shows compiler errors next to your code, saving you from having to mouse over squiggles or open the error pane all the time.
* [Code Maid](https://marketplace.visualstudio.com/items?itemName=SteveCadwallader.CodeMaidVS2022) Fixes most trivial formatting issues and saves a ton of time.
* [Conveyor by Keyoti (free version)](https://marketplace.visualstudio.com/items?itemName=vs-publisher-1448185.ConveyorbyKeyoti) This is critical to my local development work for Xamarin / Maui apps, so those apps can hit APIs on my machine. For that work I haven't needed the paid tier, but it looks like there are some nice features there.
* [Switch Startup Project](https://marketplace.visualstudio.com/items?itemName=vs-publisher-141975.SwitchStartupProjectForVS2022) This lets you save a configurations for when you want multiple startup projects. If you need to switch between groups of projects much, its a lot nicer than the visual studio option.
* [CodeRush (paid only)](https://marketplace.visualstudio.com/items?itemName=DevExpress.CodeRushforRoslyn) This is fairly inexpensive and provides some nice refactorings (particularly the ones using caps lock as a modifier). Adding images and some highlighting to comments are nice as well.

### Visual Studio Code Extensions: 

I mostly use VS Code for everything VS isn't good at handling. For me that usually means working with markdown. 

* [Markdown Paste](https://marketplace.visualstudio.com/items?itemName=telesoho.vscode-markdown-paste-image) Lets you paste an image from your clipboard directly into markdown and control where in your repo it is saved. This solved a long running problem for me of typos in image names (particularly related to case sensitivity) which would break docs on the web that otherwise worked locally.
* [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) I rely on the table of contents editor here.
* [Markdown Editor](https://marketplace.visualstudio.com/items?itemName=zaaack.markdown-editor) I use this occasionally for the WYSIWYG editor for markdown tables, where it is very helpful.
* [Spell Right](https://marketplace.visualstudio.com/items?itemName=ban.spellright) This is a well behaved spell checker. For most systems it uses your OS spelling, but understands things like pascal casing and can store custom dictionary entries in your repo. 
* [Poor Man's T-SQL Formatter](https://marketplace.visualstudio.com/items?itemName=TaoKlerks.poor-mans-t-sql-formatter-vscode) There are a bunch of flavors of this (for VS, SSMS, Web...) and I use this one. Its a huge time saver, particularly when I'm trying to understand code that isn't formatted.

### DotNet NuGet Libraries and Tools

* **General Purpose**
  * [Radzen.Blazor](https://blazor.radzen.com/) This is a nice collection of free, Blazor native UI components. Because they rely on Blazor under the hood, the change detection / rendering of Blazor behaves well, unlike similar component collections which are primarily java script based and need change notifications to redraw their UIs.
  * [Blazored Libraries](https://github.com/Blazored). In particular the Modal and Toast libraries. I particularly like using the Modal dialog interface they provide. It has allowed me to unit test UI interactions, like simulating a user hit Ok or Cancel, without actually rendering anything.
  * [stateless](https://github.com/dotnet-state-machine/stateless) Simple state machines which I like for complex workflows. You can [really simplify some Blazor UI with this](/blog/stateless-blazor-easy-integration-of-ui-and-business-logic).
  * [Humanizer](https://github.com/Humanizr/Humanizer) I started using this for the Enum rendering, but have found it covers a ton areas that let you have a better UX without writing a bunch of tedious code.
  * [OneOf](https://github.com/mcintyre321/OneOf) F# style discriminated unions in C#. I really like compile time checking that I am handling all possible responses from a method. It has also helped me reduce the use of exceptions to control flow. Use their Source Generator package if you start making using of the base class. 
* **Analyzers & Code Quality** For .Net 5+ projects, above the standard .NET analyzers:
  * [StyleCop.Analyzers](https://github.com/DotNetAnalyzers/StyleCopAnalyzers) - classic code style checker.
  * [SonarAnalyzer.CSharp](https://www.nuget.org/packages/SonarAnalyzer.CSharp/) - Provides a wide range of insightful analyzer output.
* **Unit Testing**
  * [Verify](https://github.com/VerifyTests/Verify) Very helpful for testing complicated data and source generation. I wasn't sold on this till I heard this discussion on [The Unhandled Exception Podcast](https://pca.st/fm72bakr).
  * [Stryker](https://github.com/stryker-mutator/stryker-net) This is a DotNet tool, not a NuGet library. I'm new to this tool but it provides an automated cross check to look at the quality of your unit tests and I have been impressed with the feedback it provides. More than that, it has helped me find gaps in my testing which I didn't expect. I'm particularly interested in how this can help with code review, because you can't easily tell during review if the red-green-refactor pattern was followed or if code coverage is actually meaningful coverage. Stryker can help you look at both those areas.
* **Source Generation**
  * [CommunityToolkit.Mvvm (>v8.0.0-preview3)](https://www.nuget.org/packages/CommunityToolkit.Mvvm) Generate your Xamarin Forms boilerplate code. Its in preview but working really nicely.

