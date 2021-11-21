---
layout: single
classes: wide
title:  "Simplifying C# Source Generation"
date:   2021-08-31 09:00:00 -0400
permalink: blog/2021/simplifying-csharp-source-generation
---

I have been thinking a lot about source generation in recent weeks. Its promise of saving developers from some redundant coding is challenged by its complexity. With source generators and Roslyn, you can generate almost anything, but getting started and writing generator code isn't simple. And if you wanted to create a generator, you would need to know exactly what code your users want to generate. This may be easy within a team, but as an open source project it could be very challenging.

Additionally, there are use cases for generators (`INotifyPropertyChanged` being the classic example) are just not that complicated. For many of these use cases, I don't think what we really want is to have that be its own dedicated library. Instead, I think we can offer a developer the ability to define what code they want to generate, without having to actually write a generator themselves.

In this post, I want to show a prototype of a library (which I'm tentatively calling `Gobie`), to demonstrate how this might work. We won't look at all into how the generator works here, just what the experience would be like for a developer using the generator, with two goals in mind:

1. We need it to be simple to define what should be generated.
2. It needs to be possible understand how specific code was generated.

## Goal

So, lets take the example of this `Author` class, where we want to auto generate encapsulated collection code by using an attribute on a private field. 

``` csharp
namespace ConsoleClient.Models
{
    public partial class Author
    {
        [EncapulatedCollection(CustomValidator = nameof(ValidateBooks))]
        private List<string> books = new();

        [EncapulatedCollection]
        private List<string> publishers = new();

        public bool ValidateBooks(string a)
        {
            return true;
        }
    }
}
```

And we want to generate the code below. Notice we are controlling access to the private collection and in one case applying validation logic before modifying the collection. 

``` csharp
namespace ConsoleClient.Models
{
    public partial class Author
    {
        public IEnumerable<string> Books => books.AsReadOnly();
        public IEnumerable<int> BooksLengths => books.Select(x => x.Length);
        public bool TryAddBooks(string s)
        {
            if (ValidateBooks(s))
            {
                books.Add(s);
                return true;
            }

            return false;
        }

        public IEnumerable<string> Publishers => publishers.AsReadOnly();
        public IEnumerable<int> PublishersLengths => publishers.Select(x => x.Length);
        public bool TryAddPublishers(string s)
        {
            publishers.Add(s);
            return true;
        }
    }
}
```

## Defining A Generator

In the prototype [Moustache Templates](https://mustache.github.io/mustache.5.html) are used to define what code should be generated. Generating the code above requires just a few steps:

1. The user creates their own attribute which inherits from `GobieFieldGeneratorAttribute`, which marks this attribute for use by the generator.
2. Then adds one or more template strings, an mark them with `GobieTemplate`
3. Then adds a public property `CustomValidator` so a validator method can be supplied. The generator will find `CustomValidator` and use it when rendering the template.

``` csharp
public class EncapulatedCollectionAttribute : GobieFieldGeneratorAttribute
{
    [GobieTemplate]
    private const string EncapsulationTemplate =
@"      public IEnumerable<string> {{ Property }} => {{ field }}.AsReadOnly();

    public IEnumerable<int> {{ Property }}Lengths => {{ field }}.Select(x => x.Length);
";

    [GobieTemplate]
    private const string AddMethod =
@"      public void Add{{ Property }}(string s)
    {
        {{#CustomValidator}}
        if({{CustomValidator}}(s))
        {
            {{ field }}.Add(s);
        }
        {{/CustomValidator}}
        {{^CustomValidator}}
        {{ field }}.Add(s);
        {{/CustomValidator}}
    }";

    public string CustomValidator { get; set; } = null;
}
```

As the dev is creating and updating their templates, they can see in real time how changes to the templates impact the output. Here we see a refactoring of the `Add` method template to define a  `TryAdd` instead.

![](/images/2021/simplifying-source-generation/template-update.gif)

## Debugging The Generator

It will be very important that a tool like this can help the user with debugging. We have the normal options for Analyzer projects, like raising compiler warnings if the attributes are setup incorrectly or offering code fixes.  

Additionally, because we are outputting text, we can do something like what is shown below. Here we show the dictionary of strings `Moustache` is using along with the template, to help the dev understand why they are not getting the expected output. In this example we see the template initially used `validator` when the proper key was `CustomValidator`

![](/images/2021/simplifying-source-generation/debug-template-issue.gif)

## More Possibilities

I think this approach can be taken much further than what is shown, and it isn't obvious where the limits will be. To be more specific, it isn't obvious what can be done while also keeping the API for the generator *simple*. Thinking through that more is my next goal.
