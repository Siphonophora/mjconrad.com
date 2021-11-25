---
layout: single
classes: wide
title:  "Experimenting With C# Source Generators - 1 Encapsulated Collections"
date:   2020-10-12 09:00:00 -0400
permalink: blog/experimenting-with-csharp-source-generators-1-encapulated-collections
header:
  teaser: /images/2020/exceptions-in-background-service-workers/teaser-500x300.png
---

Source Generators are a work in progress feature for Roslyn that will allow automatic generation of C# code based on existing code, or external files. From a design perspective, the most important thing to know is the generated source is *additive* only. So this doesn't provide a direct replacement for Fody or PostSharp. If source generators are totally new to you, you can take a look at [the specification](https://github.com/dotnet/roslyn/blob/master/docs/features/source-generators.md) and [some examples](https://github.com/dotnet/roslyn-sdk/tree/master/samples/CSharp/SourceGenerators) from the Roslyn team.

Because this feature is in development as of early Oct 2020, here is a quick rundown of what it took to get this working. Happily its now very easy to get started. I didn't install any preview or release candidates.
* Microsoft Visual Studio Community 2019 - Version 16.7.4. which is a current release. *Over time tooling improvements should become available first in preview versions, but its possible now to use a normal up to date install.*
* The csproj file specifies `netstandard2.0` and LangVersion `8.0`. I don't have any .Net 5 release candidates installed.

## Hello World

The Roslyn team took care of this in [the examples](https://github.com/dotnet/roslyn-sdk/tree/master/samples/CSharp/SourceGenerators), so I won't re-invent that here. If we simplify that a bit, we get the simplest possible example, where we take a string, and add it to the compilation. We can then rely on the generated class in our code, and can call it normally. 

``` csharp
    [Generator]
    public class HelloWorldGenerator : ISourceGenerator
    {
        public void Execute(GeneratorExecutionContext context)
        {
            // begin creating the source we'll inject into the users compilation
            StringBuilder sourceBuilder = new StringBuilder(@"
using System;
namespace HelloWorldGenerated
{
    public static class HelloWorld
    {
        public static void SayHello() 
        {
            Console.WriteLine(""Hello from generated code!"");
        }
    }
}");

            // inject the created source into the users compilation
            context.AddSource(
                "helloWorldGenerated", 
                SourceText.From(sourceBuilder.ToString(), Encoding.UTF8));
        }

        public void Initialize(GeneratorInitializationContext context)
        {
            // No initialization required
        }
    }
```

``` csharp
HelloWorldGenerated.HelloWorld.SayHello();
```

For me, the examples project built and ran without issue. Moreover, they provided a nice diverse set of examples to start from.

## Encapsulated Collections

I have been working a lot recently with encapsulated collections, inspired by a [post by Steve Smith](https://ardalis.com/encapsulated-collections-in-entity-framework-core/). The idea of an encapsulated collection is that we may not want to expose a list as a property, because it would allow any other class to manipulate the list through its public functions. What we can do instead is to have a private list, which is available publicly as a readonly enumerable. Then we can provide methods like `Add` which can implement additional business logic to determine whether `Add` is allowed, before the underlying collection is modified. 

This approach requires a decent bit of boilerplate, so it looked like a good candidate for generation. My goal implementation would involve the programmer adding an attribute called `EncapsulatedCollection` to their private list, and that would trigger generation of the public readonly enumerable and `Add` methods.

``` csharp
public partial class ExampleCollectionModel
{
    [EncapsulatedCollection]
    private List<string> names = new List<string>();
}
```

By cribbing off the examples, this turned out to be fairly simple, so I added another feature, which was to allow the programmer to provide validation logic that would control whether the `Add` happened, and provide the ability to specify a side effect to occur only if `Add` was allowed. I considered a few approaches, but the one that seemed like it would be most discoverable for the programmer would be to provide a private `Validator` sub-class with virtual methods that they could override, as needed.

So, if the programmer supplies the following code.

``` csharp
public partial class ExampleCollectionModel
{
    [EncapsulatedCollection]
    private List<string> names = new List<string>();

    private class CustomValidator : Validator
    {
        public override bool AddNamePermitted(string item) => item.Length % 2 == 0;

        public override void AddNameSideEffect(string item)
        {
            Console.WriteLine($"Added {item}");
        }
    }
}
```

And the generator supplies this code (shown after letting CodeMaid clean up the formatting).

``` csharp
using System.Collections.Generic;

namespace GeneratedDemo
{
    public partial class ExampleCollectionModel
    {
        private readonly Validator validator = 
            new GeneratedDemo.ExampleCollectionModel.CustomValidator();

        public IEnumerable<System.String> Names => names.AsReadOnly();

        public void AddName(System.String item)
        {
            if (this.validator.AddNamePermitted(item))
            {
                names.Add(item);
                this.validator.AddNameSideEffect(item);
            }
        }

        private class Validator
        {
            public virtual bool AddNamePermitted(string item) => false;

            public virtual void AddNameSideEffect(string item) { }
        }
    }
}
```
``` csharp
using System;
namespace EncapsulatedCollection
{
    [AttributeUsage(AttributeTargets.Field, Inherited = false, AllowMultiple = false)]
    sealed class EncapsulatedCollectionAttribute : Attribute
    {
        public EncapsulatedCollectionAttribute()
        {
        }
    }
}
```

Notice that in addition to the private class, we are also adding the attribute into the project. This is important because, like an analyzer project, the generator is part of our tooling and won't actually be deployed. Our code is using the generator, but isn't taking a runtime dependency on it.

The full [source generator code is available](https://gist.github.com/Siphonophora/54a38ad8d0034d01ee434f37580e9e6a), but please keep in mind, that its a mess and just proof of concept code.

## Source Generation Workflow

The most interesting thing in this experiment turned out to be how the validator is working. I considered a few options for how to accomplish this, and ended up deciding on an abstract class because overriding existing methods should be more discoverable than if I made a programmer follow a naming convention or something (I'm still not satisfied with this, because the `AddNamePermitted` method might need access to private fields to determine if adding is allowed).

Conceptually, I struggled the most with how how to actually use, at generation time, a class which is derived from one I am generating. **This seems like a catch-22, doesn't it?** So we somehow need to detect if the programmer using our generator has inherited from the class we are adding to the compilation. If this doesn't seem a bit strange, look again at the two halves of the partial class above. They depend *on each other*.

![](/images/2020/experimenting-with-source-generators-1/circular-dependency.png)

It turns out its the two passes involved in source generation actually allow this to work quite easily.
1. The first pass is parsing the C# and finding all the symbols, but not actually compiling them. This is one of the inputs (along with additional files provided in the csproj) which we can use when generating our source. Using the symbols Roslyn found, we can look for a class which inherits from our `Validator` by type instead of by name. And if we find one, then we can use it in our code.
   1. Following the first pass, the generator runs and produces its half of the partial class, which actually creates that `Validator`.
2. In the second pass, the compilation actually runs. Because all parts of the partial class exist, everything compiles just fine.

I suspect these two passes will allow for some really unusual workflows. For example, you might be able to have the user override your method, but providing additional parameters. When the generator runs, it could 'fix' the `Validator` virtual method to align with with the user provided override. This turns inheritance upside down, so it seems like a bad idea. But I think you might be able to accomplish it. The main reason I am pointing this out is because best practices and design patterns are still in development for source generators, so its a good time to start thinking about approaches like this.

## User Experience

Neither the experience of creating a generator, or using it, are finished. This makes the actual experimentation a bit frustrating. 

* Generated files do not yet get written to predictable locations. So, I ended up having the generator spit out a file with the generated source to a temp directory, just so I could see it.
* Thus far, I haven't figured out how to actually debug the generator while its running, so I was similarly writing debug output to myself in comments, in the generated source. 
* Finally, while consuming the generator, visual studio wasn't able to resolve the existence of the `Validator` class, so overriding its methods required that I knew what they were called. Visual Studio actually generates errors saying these are missing, but the build will still succeed.

All of these look to be on the road map for source generators before they are released. And the catch-22 above will actually be simplified in a way. In the future we should be able to register our generator to watch for the the addition of our `EncapsulatedCollection` attribute. When we see that, we could generate or update the `Validator` class in near real time. That way, the visual studio will be able to help the user create overrides for the validator methods. 

Overall, I'm excited to see more of what is possible with source generation. 