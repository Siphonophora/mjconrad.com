---
layout: single
classes: wide
title:  "CS8618, Non-Nullable Parameters, and EditorRequired in Blazor"
date:   2022-01-05 09:00:00 -0400
permalink: blog/non-nullable-parameters-editorrequired-blazor
header:
  teaser: /images/2022/non-nullable-parameters-editorrequired-blazor/teaser-500x300.png
---

Using [nullable reference types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references) with Blazor can result in many CS8618 warnings from the complier, complaining that a component `Parameter` may be null. But that often doesn't align with how we use (or want to use) our components. 

Note that this post does not deal with parameters which are supplied in URL routes.

## Quieting CS8618

There are several ways we can get rid of `CS8618` aside from declaring our properties as nullable. Of these options, the tersest general purpose option is shown below.

``` csharp
[Parameter] public string Foo { get; set; } = null!;
```

A more complete list is available on [J.Sakamoto's blog](https://dev.to/j_sakamoto/how-to-shut-the-warning-up-when-using-the-combination-of-blazor-code-behind-property-injection-and-c-nullable-reference-types-2opm).

If all you want is to get rid of the warnings, that will cover you. But, I assume you are using nullable for a reason, so lets get into what else we can do, so we aren't just ignoring Roslyn and the possibility that these parameters could be null.

## API Clarity and preserving the spirit of `nullable`

Starting with .Net 6 we have a new option to help us, which is the `EditorRequiredAttribute`. 

This will let us keep a lot of the value of nullable, and a lot of the safety, without writing null checks everywhere in our components. Additionally, it can help make proper use of components clearer. Consider the two options below. If we did the strictly proper thing as with `RequiredString1`, marking it as nullable and doing null checks in our code, `string?` still implies it is optional. `RequiredString2` is much clearer that you need to provide the string.

``` csharp
[Parameter] public string? RequiredString1 { get; set; }

[Parameter, EditorRequired] public string RequiredString2 { get; set; } = null!;
```

`EditorRequired` brings with it a new build warning `RZ2012` when parameters are definitely missing. Given that we intend for `EditorRequired` to be mandatory, it makes sense to promote `RZ2012` to be a build *error*, as shown below. 
![](/images/2022/non-nullable-parameters-editorrequired-blazor/warnings-as-errors.png)

Now we will get feedback if the parameter is definitely not provided. It doesn't cover all cases, so be sure to review the [limitations of RZ2012](#limitations---rz2012) below.

With a simple example component, we can see we get warnings and build errors when we would expect them. Importantly this applies to both standard parameters and things like render fragments, which are defined as children of the components in the razor file.

``` csharp
// ExampleComponent.razor
@code {
    [Parameter] public string? OptionalString { get; set; }

    [Parameter, EditorRequired] public string RequiredString { get; set; } = null!;

    [Parameter, EditorRequired] public RenderFragment ChildComponent { get; set; } = null!;
}
```

![](/images/2022/non-nullable-parameters-editorrequired-blazor/rz2012-use.png)

## Limitations - RZ2012

The docs for `RZ2012` are clear that there isn't a guarantee that these parameters won't be null. The code analyzer which produces this warning is pretty simple [source is here](https://github.com/dotnet/razor-compiler/blob/07e115540fd82508bf261ea85adc55734b12cdb9/src/Microsoft.AspNetCore.Razor.Language/src/Components/ComponentLoweringPass.cs). Practically, all it is looking for is a declaration for the parameter, but not checking what value is provided. Additionally, if you use [splatting](https://docs.microsoft.com/en-us/aspnet/core/blazor/components/?view=aspnetcore-6.0#attribute-splatting-and-arbitrary-parameters) it doesn't attempt to check whether parameters are populated (which seems reasonable).

Here are all the cases I could find where you won't get a diagnostic, but might expect one.

``` html
<ExampleComponent RequiredString="@nullString">
    <h1>Content</h1>
</ExampleComponent>

<ExampleComponent RequiredString=null>
    <h1>Content</h1>
</ExampleComponent>

<ExampleComponent RequiredString="Hello">
    @nullFragment
</ExampleComponent>

@* Here we 'splat' the component, providing extra parameters
   through a dictionary. *@
<ExampleComponent  @attributes=@args />
<ExampleComponent  @attributes=null />

@code{
    string? nullString = null;
    RenderFragment? nullFragment = null;
    Dictionary<string, object> args = new();
}
```

So, we can see that `RZ2012` is helpful, but it clearly is missing some more and less obvious cases. **Even when `RZ2012` is an error, there will still be a role for code review. The possible mismatch between properties that shouldn't be null and arguments that could be look to me like the easiest of these to overlook.** Even so, I think this approach should be helpful, especially if you don't make much use of splatting.

## Limitations - Cascading Parameters

`EditorRequired` doesn't work with Cascading Parameters, it says that in the docs, but you won't get a warning if you try to use them together. I think it makes sense that it doesn't, because any analyzer attempting to check for missing cascading parameters will likely be even more limited than the one for regular Parameters. 

Given that we can't get build errors related to missing cascading parameters, I think it makes sense to avoid using the `= null!;` trick shown above to silence CS8618. That leaves us with two other decent options.

* Use regular nullable types, and perform null checks as needed.
* Use value types (so long as you are happy to get the `default` for that type if the cascading parameter doesn't exist).

## Conclusion

The approach shown above can help reduce nullability issues in Blazor components without a large increase in the risk of creating null reference exceptions. There are several limitations, so care is still needed, and this approach is probably best within a team who understands those limitations. 

On the other hand, this may not be a good fit for a public component library that needs to support many different coding styles. In this case `EditorRequired` is still helpful, but you probably need more null checks in your component as a whole or in `OnParametersSet{Async}`.