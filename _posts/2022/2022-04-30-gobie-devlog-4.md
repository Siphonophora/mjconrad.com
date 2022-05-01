---
layout: single
classes: wide
title:  "Gobie DevLog 4 - Bringing It All Together"
date:   2022-04-30 12:00:00 -0400
permalink: blog/2022/gobie-devlog-4
---

I'm very happy to say that the last of the generation capabilities I want to offer in the initial release of Gobie has been added to the alpha release.

With that in place, I want to walk you through a source generator I wrote at work, and how it can be replicated more simply with Gobie.

[![NuGet](https://shields.io/nuget/v/Gobie.svg)](https://www.nuget.org/packages/Gobie/)

# Updates

Released in v0.5.0-alpha:
* Added global file templates.

# Current Status

Core functionality is all roughed out now. Currently this isn't stable and there are quite a few bugs I'm aware, of. 

However, I think this is in a good place to get community feedback about the concept and API.

# Demo - Background

The motivating example behind this entire project was a generator I wrote for work. The web apps I work on generally support moderate to high complexity clinical laboratory testing. So we need to handle complicated workflows and also need universal audit trails. 

One approach we have taken to meet those needs works similarly to what is shown in my [post about stateless and Blazor](/blog/stateless-blazor-easy-integration-of-ui-and-business-logic). We use the `Stateless` library to define a state machine which controls the lab workflow, and every time a step is taken in the workflow we capture that in a log class which is tied to whatever entity we are working with. So every entity we want to use this pattern requires the two classes below.

* `Entity` Contains the state machine, a list of log messages, and all entity data.
  * `EntityStateLog` Contains timestamp, user, comments, state and trigger for every change to the entity.

As it turned out, creating this pair through inheritance turned out to be impossible (It turned out it would have needed both [Covariance and Contravariance](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/) which can't be done in C#). That is what lead us use source generation. 

The generator we created did the following:
1. It created a marker attribute with a few parameters to gather the names of the state and trigger enums that `stateless` would use.
2. When it found that marker attribute on a partial class, the generator would:
   1. Add a bunch of boilerplate code and some partial methods to the marked class.
   2. Build the related `StateLog` class.
   3. Add some code for `EntityFrameworkCore`, so it understood how we wanted it to work with the classes.

**As it turns out, all of this was accomplished by plugging data from the marker into some code templates we provided.** The generator saved a lot of effort in the project where we used it but there were two clear drawbacks. 
* There is a steep learning curve to writing a generator, so getting other developers up to speed was expensive.
* Adding functionality to the generator was still fairly tedious, and I avoided doing so in several places where generation could help, but wouldn't be a net time save. The encapsulated collection generator shown below is one such example.

Its the heavy use of templates plus these drawbacks that lead to this library.

# Demo - Implementation

Here we will look at implementing the functionality above, but I won't spend much more time on why we are generating this specific code. Instead, I want to focus on **how** this is done.

In a several places I have shortened code for legibility. The full source for the templates shown below
[is available here](https://github.com/Siphonophora/Gobie/blob/459e1177c8cf3a77f1e739e3a47fe63a3aebf830/src/Gobie/ConsoleClient/CompleteExample.cs).

## Define the Marker Attribute

First of all, we need a marker attribute for our partial classes that will capture the names of the two enums we need for `stateless`. So we can do that by inheriting `GobieClassGenerator` and defining the two required parameters. 

``` csharp
public sealed class StatefullLoggedClassGenerator : GobieClassGenerator
{
    [Required]
    public string StateEnum { get; set; }

    [Required]
    public string TriggerEnum { get; set; }
}
```    

In response Gobie creates the attribute below.

``` csharp
public sealed class StatefullLoggedClassAttribute : global::Gobie.GobieClassGeneratorAttribute
{
    public StatefullLoggedClassAttribute(string stateEnum, string triggerEnum)
    {
        this.StateEnum = stateEnum;
        this.TriggerEnum = triggerEnum;
    }

    public string StateEnum { get; }

    public string TriggerEnum { get; }
}
```

Which we can use as shown here.

``` csharp 
    [StatefullLoggedClass(nameof(AuthorState), nameof(AuthorTrigger))]
    public partial class Author
    {
    }
```    

## Adding Code to our Partial Class

Now we need to add some functionality to our partial `Author` class, which we can do with the `GobieTemplate` below. Notice that the template us using the two strings we gather from the marker attribute `StateEnum` and `TriggerEnum`. Additionally we are using the `ClassName` string which is provided automatically by Gobie.

{% raw %}
``` csharp
public sealed class StatefullLoggedClassGenerator : GobieClassGenerator
{
    // ...

    [GobieTemplate]
    private const string StatefullLoggedClass = @"
    // Shortened for clarity....

    private readonly List<{{ClassName}}StateLog> stateLog = new List<{{ClassName}}StateLog>();

    private static partial {{StateEnum}} GetInitialState();

    /// <summary>
    /// Builds a new instance of the state machine.
    /// </summary>
    public partial StateMachine<{{StateEnum}}, {{TriggerEnum}}> GetStateMachine(
            Func<{{StateEnum}}> stateAccessor,
            Action<{{StateEnum}}> stateMutator);
    ";
}
```        

We now get the following code generated for the `Author` class. The full example generates a bunch of additional code to actually use the state machine, but what I wanted to highlight here in particular are the partial methods. By including these in the template we can require that the user supply implementations for these, just like we would if we were using `abstract` with inheritance.

``` csharp
public partial class Author
{
    // Shortened for clarity....

    private readonly List<AuthorStateLog> stateLog = new List<AuthorStateLog>();

    private static partial AuthorState GetInitialState();

    /// <summary>
    /// Builds a new instance of the state machine.
    /// </summary>
    public partial StateMachine<AuthorState, AuthorTrigger> GetStateMachine(Func<AuthorState> stateAccessor, Action<AuthorState> stateMutator);
}
```

## Creating the Log Class

In the code above we are referencing a class `AuthorStateLog` which we want to generate. If we wanted the log to be a nested class, we could have done that in the template above. Thats because the standard `GobieTemplate` puts generated code inside a partial class declaration. 

In this case we have decided against nesting, so we can use a `GobieFileTemplate` which will allow us to define the complete contents of a new file to add to the compilation. Note the argument `StateLog` given to the template attribute is defining the suffix of the file name.

``` csharp
public sealed class StatefullLoggedClassGenerator : GobieClassGenerator
{
    // ...

    [GobieFileTemplate("StateLog")]
    private const string KeyString = @"
    using System;

    namespace {{ClassNamespace}};

    public sealed class {{ClassName}}StateLog
    {
        // ...

        /// <summary>
        /// Current state after the change.
        /// </summary>
        public {{StateEnum}} State { get; private set; }

        /// <summary>
        /// Trigger that fired.
        /// </summary>
        public {{TriggerEnum}}? FiredTrigger { get; private set; }
    }
    ";
}
```      

And we generate a new file with this contents:

``` csharp
using System;

namespace SomeNamespace;
public sealed class AuthorStateLog
{
    // ...

    /// <summary>
    /// Current state after the change.
    /// </summary>
    public AuthorState State { get; private set; }

    /// <summary>
    /// Trigger that fired.
    /// </summary>
    public AuthorTrigger? FiredTrigger { get; private set; }
}
```

## Setting up a Global Template

Now that we have the code generation above setup, we can apply the `StatefullLoggedClass` and get the primary functionality we want. In this example we also have several mappings we need to do for Entity Framework Core, for every instance of `StatefullLoggedClass`. And the best option would be if we could put all mapping code in a single place, regardless of how many classes we generate code for.

We can do all of this with a `GobieGlobalFileTemplate`. Just like the `GobieFileTemplate` we are defining an entire file's content but with a key difference:
* In a global template, the only Mustache variable you can use is `ChildContent`. This is the location where other generators will insert code. That code could come from anywhere in the same assembly. 
  
``` csharp
[GobieGeneratorName("EFCoreRegistrationAttribute", Namespace = "ConsoleClient")]
public sealed class EFCoreRegistrationGenerator : GobieGlobalGenerator
{
    [GobieGlobalFileTemplate("EFCoreRegistrationGenerator", "EFCoreRegistration")]
    private const string KeyString = @"
        namespace SomeNamespace;

        public sealed static class EFCoreRegistration
        {
            public static void Register(Microsoft.EntityFrameworkCore.ModelBuilder mb)
            {
                if (mb is null)
                {
                    throw new ArgumentNullException(nameof(mb));
                }

                {{ChildContent}}
            }
        }
        ";
}
```

This generator produces marker attribute for the Assembly. So we can invoke it as shown:

``` csharp
[assembly: EFCoreRegistration]
```

And this generates our global template, without anything in our `ChildContent`

``` csharp
namespace SomeNamespace;
public sealed static class EFCoreRegistration
{
    public static void Register(Microsoft.EntityFrameworkCore.ModelBuilder mb)
    {
        if (mb is null)
        {
            throw new ArgumentNullException(nameof(mb));
        }
    }
}
```

## Filling the Global Template

In order to add code to the `ChildContent` above, we define one more template in our `StatefullLoggedClassGenerator`. This acts just like the others, except that it specifies the name of the global template it should be added to.

``` csharp
public sealed class StatefullLoggedClassGenerator : GobieClassGenerator
{
    // ...

    [GobieGlobalChildTemplate("EFCoreRegistrationGenerator")]
    private const string EfCoreRegistration = @"
        // StatefullLoggedClassGenerator for {{ClassName}} - Map enums to strings and log field.
        mb.Entity<{{ClassName}}>().Property(x => x.State).HasConversion<string>();
        mb.Entity<{{ClassName}}StateLog>().Property(x => x.State).HasConversion<string>();
        mb.Entity<{{ClassName}}StateLog>().Property(x => x.FiredTrigger).HasConversion<string>();
        mb.Entity<{{ClassName}}>()
            .Metadata.FindNavigation(nameof({{ClassName}}.StateLog))?
            .SetPropertyAccessMode(PropertyAccessMode.Field);";
}
```

Just by doing that, we now have our registration code added to the global template. **Note that when multiple generators are inserting code into the global template, you don't have control over the order that code is placed in the template.** If you have a use case that would require it, let me know.

``` csharp
namespace SomeNamespace;
public sealed static class EFCoreRegistration
{
    public static void Register(Microsoft.EntityFrameworkCore.ModelBuilder mb)
    {
        if (mb is null)
        {
            throw new ArgumentNullException(nameof(mb));
        }

        // StatefullLoggedClassGenerator for Author - Map enums to strings and log field.
        mb.Entity<Author>().Property(x => x.State).HasConversion<string>();
        mb.Entity<AuthorStateLog>().Property(x => x.State).HasConversion<string>();
        mb.Entity<AuthorStateLog>().Property(x => x.FiredTrigger).HasConversion<string>();
        mb.Entity<Author>().Metadata.FindNavigation(nameof(Author.StateLog))?.SetPropertyAccessMode(PropertyAccessMode.Field);
    }
}
```

## Encapsulated Collection

This is the finishing touch, and it happens to be where I drew the line when implementing the generator at work. It would have been nice, but implementing it wouldn't have saved time.

Below we add the same logic we have seen before, but with the addition that it adds EFCore registration to our global template. 

``` csharp
public sealed class EncapsulatedCollectionGenerator : GobieFieldGenerator
{
    [GobieTemplate]
    private const string EncapsulatedCollection = @"
        public System.Collections.Generic.IEnumerable<{{FieldGenericType}}> {{FieldName : pascal}} => {{FieldName}}.AsReadOnly();

        public void Add{{ FieldName : pascal }}({{FieldGenericType}} s)
        {
            if(s is null)
            {
                throw new ArgumentNullException(nameof(s));
            }

            {{#CustomValidator}}
            if({{CustomValidator}}(s))
            {
                {{FieldName}}.Add(s);
            }
            {{/CustomValidator}}

            {{^CustomValidator}}
                {{FieldName}}.Add(s);
            {{/CustomValidator}}
        }
";

    [GobieGlobalChildTemplate("EFCoreRegistrationGenerator")]
    private const string EfCoreRegistration = @"

        // EncapsulatedCollectionGenerator for {{ClassName}} - Map the encapsulated collection
        mb.Entity<{{ClassName}}>()
            .Metadata.FindNavigation(nameof({{ClassName}}.{{FieldName : Pascal}}))?
            .SetPropertyAccessMode(PropertyAccessMode.Field);

";

    [Required]
    public string CustomValidator { get; set; } = null;
}
```   

{% endraw %} 

# Summary

As we have shown above, we have several methods of defining code we want to generate with Gobie. Primarily this covers simple generation where we really just need to fill out a template, though more functionality may make sense to add. How that takes shape will depend on community feedback. 

Again, the full template code we used above
[is available here](https://github.com/Siphonophora/Gobie/blob/459e1177c8cf3a77f1e739e3a47fe63a3aebf830/src/Gobie/ConsoleClient/CompleteExample.cs).

# Feedback

I am very interested in feedback. Feel free to drop a note in the comments here or open an issue with your thoughts in [the project repo](https://github.com/Siphonophora/Gobie).
