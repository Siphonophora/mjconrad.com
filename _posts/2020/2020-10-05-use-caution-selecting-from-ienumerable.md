---
layout: single
classes: wide
title:  "Use Caution When Using Select with IEnumerable or IQueryable"
date:   2020-10-05 09:00:00 -0400
permalink: blog/use-caution-when-using-select-with-ienumerable-or-iqueryable
---

I was recently caught off guard by the behavior of an `IEnumerable` produced by a `Select` statement. The code I was using looked a bit like what is shown below.

``` csharp
public void ProjectionExample(List<ModelClass> modelClasses)
{
    var projections = modelClasses.Select(x => new ProjectedClass(x));
    projections.First().Name = "Foo";

    foreach (var projection in projections)
    {
        if (projection.Name == "Foo")
        {
            // This IF is never hit.
        }
    }
}
```

In the code, I used a `Select` to transform a model class into another class. Then I made changes to some members of the projected class (line 4) and was surprised to find that those changes where not present when looping through the collection. As a result an if (line 8) condition was never met, and some unit tests were failing. Lets take a look at what was happening. 

``` csharp
// https://source.dot.net/#System.Linq/System/Linq/Select.cs

private sealed partial class SelectIListIterator<TSource, TResult> : Iterator<TResult>
{
    public SelectIListIterator(IList<TSource> source, Func<TSource, TResult> selector)
    {
        ...
    }
    
    public override bool MoveNext()
    {
         ...
        if (_enumerator.MoveNext())
        {
            _current = _selector(_enumerator.Current);
            return true;
        }
        ...
    }     
    
    ...
}
```

A simplified version of the source code for `Select` [source.dot.net](https://source.dot.net) is shown above. In out example we select from a list, so we are interested in the `SelectIListIterator.MoveNext()` method. This is the method being called **every time** we access the contents of `projections`. It doesn't matter whether if we are looping, filtering, calling `First` or `Take`.... We always use `MoveNext()`. 

In turn, `MoveNext()` calls the Func `selector` to produce the value. In our case, this was `x => new ProjectedClass(x)`. Maybe the problem with the code above is jumping out at you now. The `First` and  `foreach` are calling `MoveNext()` independently and as a result *they create different instances of `ProjectedClass`*. This explains why changing the Name property didn't seem to be permanent.

This is what is meant by [deferred execution](https://docs.microsoft.com/en-us/dotnet/standard/linq/deferred-execution-lazy-evaluation) and we need to be particularly careful when using it with `Select`. 

We can extend this with a quick example. Here, we again produce a collection (with only one member) called `projections` which is an `IEnumerable` that will transform one type into another. We then test the `IEnumerable` and several other collection types to see if they use deferred or eager execution. 

``` csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace SelectEnumerable
{
    internal class Program
    {
        private static void Main(string[] args)
        {
            var sourceInts = new List<IntHolder> { new IntHolder { Value = 1 } };

            var projections = sourceInts.Select(x =>
            {
                Console.WriteLine("Projecting IntHolder to StrHolder");
                return new StrHolder { Value = x.Value.ToString() };
            });

            TestCollection("IEnumerable", () => projections);
            TestCollection("IEnumerable (with Where)", () => projections.Where(x => true));
            TestCollection("IEnumerable (with Skip)", () => projections.Skip(0));
            TestCollection("IQueryable", () => projections.AsQueryable());
            TestCollection("Array", () => projections.ToArray());
            TestCollection("HashSet", () => projections.ToHashSet());
            TestCollection("List", () => projections.ToList());
            TestCollection("IEnumerable, produced from intermediate List", () => projections.ToList().AsEnumerable());
            TestCollection("IEnumerable, produced by filtering an intermediate List", () => projections.ToList().Where(x => true));
        }

        private static void TestCollection(string typeName, Func<IEnumerable<StrHolder>> func)
        {
            Console.WriteLine($"{typeName}:");
            var projections = func();
            var a = projections.First();
            var b = projections.First();
            if (a == b)
            {
                Console.WriteLine($"{typeName} used Eager Execution on the Select.");
            }
            else
            {
                Console.WriteLine($"{typeName} used Defered Execution on the Select.");
            }
            Console.WriteLine();
        }
    }

    internal class IntHolder { public int Value { get; set; } }

    internal class StrHolder { public string Value { get; set; } }
}
```

This console output produced is shown below. 

```
IEnumerable:
Projecting IntHolder to StrHolder
Projecting IntHolder to StrHolder
IEnumerable used Defered Execution on the Select.

IEnumerable (with Where):
Projecting IntHolder to StrHolder
Projecting IntHolder to StrHolder
IEnumerable (with Where) used Defered Execution on the Select.

IEnumerable (with Skip):
Projecting IntHolder to StrHolder
Projecting IntHolder to StrHolder
IEnumerable (with Skip) used Defered Execution on the Select.

IQueryable:
Projecting IntHolder to StrHolder
Projecting IntHolder to StrHolder
IQueryable used Defered Execution on the Select.

Array:
Projecting IntHolder to StrHolder
Array used Eager Execution on the Select.

HashSet:
Projecting IntHolder to StrHolder
HashSet used Eager Execution on the Select.

List:
Projecting IntHolder to StrHolder
List used Eager Execution on the Select.

IEnumerable, produced from intermediate List:
Projecting IntHolder to StrHolder
IEnumerable, produced from intermediate List used Eager Execution on the Select.

IEnumerable, produced by filtering an intermediate List:
Projecting IntHolder to StrHolder
IEnumerable, produced by filtering an intermediate List used Eager Execution on the Select.
```

As we can see, the methods `ToArray`, `ToList` and `ToHashSet` all perform eager execution when creating their target collection type. As a result, when we use any of these **after** our `Select` statement then each instance of our `ProjetedClass` is created one time. Therefore, creating a `List`, `Array` or `HashSet` will make your code easier to correctly reason about, which in turn swill mean it is more likely to  be correct. The only exception that jumps out at me is a loop like this `foreach (var item in sourceInts.Select(x => new StrHolder(x)){ ... }` because the `Select` is guaranteed to only be iterated over once.