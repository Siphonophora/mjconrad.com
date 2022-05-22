---
layout: single
classes: wide
title:  "Gobie DevLog 3 - Simple C# Source Generation"
date:   2022-04-17 12:00:00 -0400
permalink: blog/2022/gobie-devlog-3
---

In [the last post about Gobie](/blog/2021/gobie-devlog-2) we looked at Class and Field templates along with formatting options. Today we can look at File templates.

[![NuGet](https://shields.io/nuget/v/Gobie.svg)](https://www.nuget.org/packages/Gobie/)

# Updates

Released in v0.4.0-alpha:
* Added file templates.

# Current Status

Just to be clear, this is currently not usable for anything other than a demo. Additionally, it isn't stable.

# Demo

File templates, the ability to define that a separate file (and probably class) should be generated, is the third of four core features I wanted to include in Gobie on initial release. The usage for this pattern which is clearest to me, is to generate a dedicate log class for your existing class. Below we will look at a simple example of this pattern. In fact it is simple enough we could rely on inheritance and not generation for the bulk of the log class. In a future post we will look at a more complex example.

{% raw %}
``` csharp
    [GobieGeneratorName("LoggedClass")]
    public sealed class LoggedClassGenBase : GobieClassGenerator
    {
        [GobieTemplate]
        private const string LogString = @"
            private readonly System.Collections.Generic.List<{{ClassName}}Log> logs = new();

            /// <summary> Standardized log entries for <see cref=""{{ClassName}}"">. </summary>
            public System.Collections.Generic.IEnumerable<{{ClassName}}Log> Logs => logs.AsReadOnly();";

        [GobieFileTemplate("Log")]
        private const string KeyString = @"
            using System;

            namespace {{ClassNamespace}};

            public sealed class {{ClassName}}Log
            {
                public int Id { get; set; }

                public {{ClassName}} Parent {get; set;}

                public DateTime Timestamp {get; set;}

                public string LogMessage {get; set;}
            }
            ";
    }
```
{% endraw %}

This is the first time we need to have multiple templates in a generator. 
* `GobieTemplate` is defining the code we will add to the existing partial class.
* `GobieFileTemplate` defines a new file and new class we will generate. In the case of the `Author` class, this will generate an `AuthorLog` class.

We can apply the template as shown below.

``` csharp
namespace Models
{
    [LoggedClass]
    public partial class Author
    {
    }
}
```

And we generate the two files below:

``` csharp
// File: Author_LoggedClassAttribute.g.cs
namespace Models
{
    public partial class Author
    {
        private readonly System.Collections.Generic.List<AuthorLog> logs = new();
        /// <summary> Standardized log entries for <see cref = "Author">. </summary>
        public System.Collections.Generic.IEnumerable<AuthorLog> Logs => logs.AsReadOnly();
    }
}
```

``` csharp
// File: Author_LoggedClassAttribute_Log.g.cs
using System;

namespace Models;
public sealed class AuthorLog
{
    public int Id { get; set; }

    public Author Parent { get; set; }

    public DateTime Timestamp { get; set; }

    public string LogMessage { get; set; }
}
```      

# Feedback

I am very interested in feedback. Feel free to drop a note in the comments here or open an issue with your thoughts in [the project repo](https://github.com/GobieGenerator/Gobie)
