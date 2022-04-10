---
layout: single
classes: wide
title:  "Gobie DevLog 2 - Simple C# Source Generation"
date:   2022-04-10 12:00:00 -0400
permalink: blog/2022/gobie-devlog-2
---

In [the last post about Gobie](/blog/2021/gobie-devlog-1) I showed the first proof of concept. Today we have a bit of an expanded feature set to show which allows for simpler template declaration.

[![NuGet](https://shields.io/nuget/v/Gobie.svg)](https://www.nuget.org/packages/Gobie/)

# Updates

* Support both Fields and Classes as generator targets.
* Provide pre-defined data to the template, like `ClassName` or `FieldName` so users don't need to provide it.
* Allow formatting options `pascal` and `camel` to be used in the templates.

# Current Status

Just to be clear, this is currently not usable for anything other than a demo. It isn't stable and the feature set is much too limited.

# Demo

Just like last time we are generating an encapsulated field, but in this case we have a more realistic syntax. The field itself is declared by the user, and annotated with our generator attribute.

We rely on two default parameters `FieldName` and `FieldGenericType` instead of explicitly providing those values. Additionally we use the 
formatting helpers like {% raw %}`{{ FieldName : pascal}}` which will take the input `books` and output `Books` in the rendered template.

{% raw %}
``` csharp
public sealed class EncapsulatedCollectionGenerator : GobieFieldGenerator
{
    [GobieTemplate]
    private const string EncapsulationTemplate =
    @"
        public System.Collections.Generic.IEnumerable<{{FieldGenericType}}> {{FieldName : pascal}} => {{FieldName : camel}}.AsReadOnly();

        public void Add{{ FieldName : pascal }}({{FieldGenericType}} s)
            {
                {{#CustomValidator}}
                if({{CustomValidator}}(s))
                {
                    {{FieldName : camel}}.Add(s);
                }
                {{/CustomValidator}}

                {{^CustomValidator}}
                    {{FieldName : camel}}.Add(s);
                {{/CustomValidator}}
            }";

    [Required]
    public string CustomValidator { get; set; } = null;
}
```
{% endraw %}

Defining the class that inherits from `GobieFieldGenerator` tells the generator to create the marker attribute `EncapsulatedCollectionAttribute` which we can use below. The usage of the attribute is shown below. 

``` csharp
public partial class Author
{
    [EncapsulatedCollection(nameof(ValidateBooks))]
    private readonly List<string> books = new();

    public bool ValidateBooks(string s) => s.Length > 0;
}
```

The generated output is shown below.

``` csharp
namespace Models
{
    public partial class Author
    {
        public System.Collections.Generic.IEnumerable<string> Books => books.AsReadOnly();
        public void AddBooks(string s)
        {
            if (ValidateBooks(s))
            {
                books.Add(s);
            }
        }
    }
}
```

# Feedback

I am very interested in feedback. Feel free to drop a note in the comments here or open an issue with your thoughts in [the project repo](https://github.com/Siphonophora/Gobie)
