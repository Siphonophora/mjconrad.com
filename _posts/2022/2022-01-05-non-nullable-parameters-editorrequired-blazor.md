---
layout: single
classes: wide
title:  "CS8618, Non-Nullable Parameters, and EditorRequired in Blazor"
date:   2022-01-05 09:00:00 -0400
permalink: blog/non-nullable-parameters-editorrequired-blazor
# header:
#   teaser: /images/2022/non-nullable-parameters-editorrequired-blazor/teaser-500x300.png
---

Using [nullable reference types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references) with Blazor can result in many CS8618 warnings from the complier, complaining that a component `Parameter` may be null. But that often doesn't align with how we use (or want to use) our components. When 

## Quieting CS8618

There are several ways we can get rid of `CS8618` aside from declaring our properties as nullable, which are covered in [J.Sakamoto's blog](https://dev.to/j_sakamoto/how-to-shut-the-warning-up-when-using-the-combination-of-blazor-code-behind-property-injection-and-c-nullable-reference-types-2opm).

Of these options, the tersest general purpose option is shown below.

``` csharp
[Parameter] public string Foo { get; set; } = null!;
```

If all you want is to get rid of the warnings, that will cover you. But, I assume you are using nullable for a reason, so lets get into what else we can do, so we aren't just ignoring Roslyn and the possibility that these parameters could be null.

## Keeping the spirit of `nullable`

Starting with .Net 6 we have another option to help us 

For internal code.

## Limitations - CascadingParameters


## Limitations - EditorRequired



Splatting.
https://docs.microsoft.com/en-us/aspnet/core/blazor/components/?view=aspnetcore-6.0#attribute-splatting-and-arbitrary-parameters



Actual diagnostic happens here:
https://github.com/dotnet/razor-compiler/blob/main/src/Microsoft.AspNetCore.Razor.Language/src/Components/ComponentLoweringPass.cs

