---
layout: single
classes: wide
title:  "Entity Framework Core Automatically Adds Missing Entity Relationships"
date:   2021-08-01 09:00:00 -0400
permalink: blog/2021/efcore-missing-relationships-1
---

Relationships in Entity Framework Core typically are represented by a pair of Navigation Properties. The parent in the relationship has a list of children, and the child has a reference to its parent. Normally setting up this kind of bi-directional relationship between entities requires that we add child to the parent collection and point the child to the parent. But when using EFCore our `DbContext` will often add a missing relationship if we create one, but not both of these references.

To look at this behavior, we will use the entities `Author` and `Book` below.

``` csharp
public class Author
{
    public int Id { get; set; }

    public string Name { get; set; }

    public List<Book> Books {get; set;} = new();

    public string AllTitles =>
        Books.Any() ?
        "Books contains " + string.Join(", ", Books.Select(x => $"'{x.Title}'")) :
        "Has no linked books.";
}

public class Book
{
    public int Id { get; set; }

    public Author Author { get; set; }

    public string Title { get; set; }
}
```

Because we are looking at behavior of the `DbContext` and not the underlying database, we can use the `Microsoft.EntityFrameworkCore.InMemory` NuGet package, which is most often used in unit testing. 

Below, we will create and track a single `Author` instance. Then we will test four variations on creating a `Book` to see how they behave.

``` csharp
private static void Main(string[] args)
{
    var dbOptions = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .EnableSensitiveDataLogging()
            .Options;

    using (var ctx = new AppDbContext(dbOptions))
    {
        var author = new Author { Name = "Terry Mancour" };
        ctx.Add(author);

        // First, lets setup all the relationships for the book Spellmonger.
        var book = new Book { Author = author, Title = "Spellmonger" };
        author.Books.Add(book);

        // Book with an Author specified. We don't add it to the Author's Book collection
        // but do add the book to dbContext tracking.
        var notLinkedToAuthor = new Book { Author = author, Title = "UnLinked" };
        ctx.Add(notLinkedToAuthor);

        // Book with an Author specified. We don't add it to the Author's Book collection or
        // add the book to dbContext tracking.
        var untracked = new Book { Author = author, Title = "Untracked" };

        // For the 'Authorless' book, we only add it to the author's book collection, but
        // leave Book.Author null.
        var authorlessBook = new Book { Title = "Authorless" };
        author.Books.Add(authorlessBook);

        Console.WriteLine("======= Before Db Save ========");
        Console.WriteLine($"{author.Name}: {author.AllTitles}");
        foreach (var b in author.Books)
        {
            Console.WriteLine($"'{b.Title}' written by {b.Author?.Name}");
        }
        ctx.SaveChanges();

        Console.WriteLine("\r\n======= After Db Save ========");
        Console.WriteLine($"{author.Name}: {author.AllTitles}");
        foreach (var b in author.Books)
        {
            Console.WriteLine($"'{b.Title}' written by {b.Author?.Name}");
        }
    }

    using (var ctx = new AppDbContext(dbOptions))
    {
        var authors = ctx.Authors.Include(x => x.Books).ToList();

        Console.WriteLine("\r\n======= Retrieved from Db ========");
        foreach (var author in authors)
        {
            Console.WriteLine($"{author.Name}: {author.AllTitles}");
        }
    }
}

// Console Output:

// ======= Before Db Save ========
// Terry Mancour: Books contains 'Spellmonger', 'UnLinked', 'Authorless'
// 'Spellmonger' written by Terry Mancour
// 'UnLinked' written by Terry Mancour
// 'Authorless' written by

// ======= After Db Save ========
// Terry Mancour: Books contains 'Spellmonger', 'UnLinked', 'Authorless'
// 'Spellmonger' written by Terry Mancour
// 'UnLinked' written by Terry Mancour
// 'Authorless' written by Terry Mancour

// ======= Retrieved from Db ========
// Terry Mancour: Books contains 'UnLinked', 'Spellmonger', 'Authorless'

```

We can see a few different types of behavior here that deserve discussion. 

* **Spellmonger** is our standard example where both sides of the relationship are setup in our code. Notice how this book becomes tracked by our `ctx` when it is added to the `Author`'s collection of books.
* **UnLinked** in itself, is a complete `Book`. It has a title and an `Author`. When we add this to our `ctx` tracking, EFCore adds the `Book` to the `Author.Books` list right away.
* **UnTracked** is identical to the example above, except we do not add it to our `ctx`. Because `ctx` has no knowledge this entity exists, it doesn't add relationships or persist it to the database.
* **Authorless** is created with only a title. When we add it to `Author.Books` our `ctx` begins tracking this `Book` but it *does not* populate the `Book.Author` property. The relationship is added only after we save to the database. 

As you can see, in both our **UnLinked** and **Authorless** examples, EFCore is adding missing relationships for us, but we need to be cautious when relying on this behavior because the two examples behave differently. Once we save to the database, the relationship between any two tracked Entities should be complete, but before saving the status of the relationships will depend on which process you followed. This feature is quite useful, but I think its worth thinking about how and when to use this automatic linking.

In the [next post](/blog/2021/efcore-missing-relationships-2), we will see additional caution is warranted, because this behavior can totally bypass business logic you may create to control when new relationships can be created.