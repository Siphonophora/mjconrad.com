---
layout: single
classes: wide
title:  "Gobie DevLog 1 - Simple C# Source Generation"
date:   2022-04-03 20:00:00 -0400
permalink: blog/2022/gobie-devlog-1
---

In [the last post about Gobie](/blog/2021/simplifying-csharp-source-generation) I outlined how a source generator which relies on user defined templates might work. In this post, I will briefly cover a proof of concept I have put together which is now available on Nuget. All we have at this point is a narrow slice of the functionality that is needed.

[![NuGet](https://shields.io/nuget/v/Gobie.svg)](https://www.nuget.org/packages/Gobie/)

# Updates

* The proof of concept code is now doing all of the intended steps of finding templates / user data and then building the templates.
* The generator is now an incremental generator. Big thanks to Andrew Lock for [his series](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/) on setting up and testing incremental generators.
* I spent a lot of time looking at how much feedback we can give to users of the library through complier warnings and errors. I think we can do quite a lot to help users define templates properly. That includes providing diagnostics if the Mustache template sytax is incorrect, or Identifiers are used in the template which are not defined. So I think even though the templates are strings, that we can give more of a compiled code feel to the warnings/errors that might occur. Some initial implementation is done in this direction, but I won't be covering it below.

# Current Status

Just to be clear, this is currently not usable for anything other than a demo. It isn't stable and the feature set is much too limited.

# Demo

This is pretty much the same as the last demo, except that this time the generator is dynamically retrieving everything. We are creating a private List and proving public methods to read/edit the list.

The first step we need is to define the Generator we want. Below we can see there are two template strings defined using Mustache like syntax. (It could be just one template if we wanted). And below that are four properties which match the identifiers used in the template strings. 

{% raw %}
``` csharp
    public sealed class EncapsulatedCollectionGenerator : GobieFieldGenerator
    {
        [GobieTemplate]
        private const string EncapsulationTemplate =
@"
            private readonly List<{{TypeName}}> {{Field}} = new List<{{TypeName}}>();
            public IEnumerable<{{TypeName}}> {{PropertyName}} => {{Field}}.AsReadOnly();
            public IEnumerable<int> {{ Property }}Lengths => {{ Field }}.Select(x => x.Length);
        ";

        [GobieTemplate]
        private const string AddMethod =
    @"      public void Add{{ Property }}({{TypeName}} s)
            {
                {{#CustomValidator}}
                if({{CustomValidator}}(s))
                {
                    {{Field}}.Add(s);
                }
                {{/CustomValidator}}

                {{^CustomValidator}}
                    {{Field}}.Add(s);
                {{/CustomValidator}}
            }";

        [Required]
        public string PropertyName { get; set; }

        [Required]
        public string Field { get; set; }

        [Required]
        public string TypeName { get; set; }

        [Required]
        public string CustomValidator { get; set; } = null;
    }
```
{% endraw %}

Defining the class that inherits from `GobieFieldGenerator` tells the generator to create the market attribute below. In this example we marked all the properties above as `Required` so they have each become constructor args. It isn't shown, but Named Parameters are used for optional properties.

``` csharp 
namespace Gobie
{
    public sealed class EncapsulatedCollectionAttribute : Gobie.GobieFieldGeneratorAttribute
    {
        public EncapsulatedCollectionAttribute(string propertyName, string field, string typeName, string customValidator = null)
        {
            this.PropertyName = propertyName;
            this.Field = field;
            this.TypeName = typeName;
            this.CustomValidator = customValidator;
        }

        public string PropertyName { get; }
        public string Field { get; }
        public string TypeName { get; }
        public string CustomValidator { get; }
    }
}
```

Now we can apply the marker attribute to a partial class and the encapsulated collection is added. 

![](/images/2022/gobie-devlog-1/gobie-devlog-1-demo.gif)

Thats it for today.

# Feedback

I am very interested in feedback. Feel free to drop a note in the comments here or open an issue with your thoughts in [the project repo](https://github.com/Siphonophora/Gobie)
