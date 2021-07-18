---
layout: single
classes: wide
title:  "Entity Framework Core Can Bypass Business Logic When Automatically Adding Missing Entity Relationships"
date:   2021-08-02 09:00:00 -0400
permalink: blog/2021/efcore-missing-relationships-2
---

As discussed in our [last post](/blog/2021/efcore-missing-relationships-1), EFCore will add missing relationships between entities when they first become tracked or when they are saved to the database (depending on how the relationship is created). This behavior can bypass business logic we might create to govern when this relationship can be created. Below we will discuss how and when this can happen, along with options to address it.

Lets take our `Author` class from last time, and add some protections to its list of `Books`. Here we have made `Books` an [encapsulated collection](https://ardalis.com/encapsulated-collections-in-entity-framework-core/). A new method `AddBookIfAllowed` will prevent a `Book` with a null or empty title from being added to our `Author`.

``` csharp
public class Author
{
    private readonly List<Book> books = new();

    public int Id { get; private set; }

    public string Name { get; set; }

    public IEnumerable<Book> Books => books.AsReadOnly();

    public string AllTitles =>
        Books.Any() ?
        "Books contains " + string.Join(", ", Books.Select(x => $"'{x.Title}'")) :
        "Has no linked books.";

    public void AddBookIfAllowed(Book book)
    {
        if (string.IsNullOrWhiteSpace(book.Title))
        {
            Console.WriteLine(
                $"Unable to add book to author {Name} with title '{book.Title}'\r\n");
            return;
        }

        books.Add(book);
    }
}

public class Book
{
    public int Id { get; private set; }

    public Author Author { get; set; }

    public string Title { get; set; }
}

    public class AppDbContext : DbContext
    {
        public AppDbContext([NotNull] DbContextOptions<AppDbContext> options)
            : base(options)
        {
        }

        public DbSet<Author> Authors { get; set; }

        public DbSet<Book> Books { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Now that we are using the encapsulated collection, we need to
            // tell EFCore to use the backing field and not the property.
            modelBuilder.Entity<Author>()
                .Metadata.FindNavigation(nameof(Author.Books))
                .SetPropertyAccessMode(PropertyAccessMode.Field);
        }
    }
```

We should take a moment to think through exactly how much protection the encapsulated collection and `AddBookIfAllowed` method can provide, within .Net. Private fields, methods and private property setters can all be [easily bypassed](https://www.concurrency.com/blog/june-2019/using-c-reflection-to-succinctly-access-private-members). Normally, we wouldn't bypass these protections in our code, because have added them to define our business logic and simplify our APIs. On the other hand an Object Relationship Mapper like Entity Framework Core or a JSON deserializer, will often be designed to bypass restrictions on setting private fields and properties because we want these tools to load external into protected fields and properties.

The most obvious example is `public int Id { get; private set; }` on both Entities are privately settable. The `Id`, as a primary key, is determined by our database, so we rely on EFCore to populate it on save. Less obvious is that EFCore reads and write to `Author.books` field when loading records, bypassing `AddBookIfAllowed`. 

So, we shouldn't be too surprised that EFCore *can* bypass our business logic. Lets take a look at *when* it will using an example book that will fail the validation on `AddBookIfAllowed`.


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
        author.AddBookIfAllowed(book);

        // Book with an Author specified, but the empty title. Our buisness logic will
        // prevent the book being added to the author. But, by adding the entity to our
        // dbContext, EF Core will bypass our business logic. This book both gets its
        // relationship built by EF Core before save, and is persisted to the database.
        var untitledBook = new Book { Author = author, Title = "   " };
        author.AddBookIfAllowed(untitledBook);
        ctx.Add(untitledBook);

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

// Unable to add book to author Terry Mancour with title '   '

// ======= Before Db Save ========
// Terry Mancour: Books contains 'Spellmonger', '   '
// 'Spellmonger' written by Terry Mancour
// '   ' written by Terry Mancour

// ======= After Db Save ========
// Terry Mancour: Books contains 'Spellmonger', '   '
// 'Spellmonger' written by Terry Mancour
// '   ' written by Terry Mancour

// ======= Retrieved from Db ========
// Terry Mancour: Books contains '   ', 'Spellmonger'
```

In the output we can see that our invalid book was rejected by `AddBookIfAllowed`. But when we add to the `ctx`, the illegal entry will still be linked to the author and persisted to the database. This is exactly the same behavior we saw in the previous post. 

## Improving our Business Model

After thinking about this for a while, I believe there is a subtle flaw in our model for `Book`, if we only intend to establish a relationship through `Author.AddBookIfAllowed`.
 
``` csharp
public class Book
{
    public int Id { get; private set; }

    public Author Author { get; set; }

    public string Title { get; set; }
}
```

That flaw, is **`Author` is publicly settable**. It isn't the public setter per se, but the fact that `Book.Author` can be set by normal code (i.e. without bypassing a private setter). A constructor for `Book` that took an `Author` parameter would have the exact same problem. The reason I see this as a flaw, is that by offering the ability to set the `Author`, EFCore immediately knows the foreign key to the `Authors` table to use when inserting our new `Book` row to the database. The end effect is identical to what would happen if we just directly inserted this new `Book` to the database. It would be there next time our app loaded data, and `AddBookIfAllowed` would never be consulted.

This can, of course, be easily fixed. 

``` csharp
public class Book
{
    public int Id { get; private set; }

    /// <summary>
    /// Author of the book. Note, the relationship between with an <see cref="Author">
    /// should be established through <see cref="Author.AddBookIfAllowed"/>.
    /// </summary> 
    public Author Author { get; private set; }

    public string Title { get; set; }
}
```

As shown above, we can simply make Author privately settable. This removes our option to bypass `AddBookIfAllowed` in normal usage. The biggest remaining risk I see is forgetting *why* `Author` isn't publicly settable. So, **I think the comment here is important to prevent ourselves or another developer from removing this protection down the road**. If you can think of something more robust, please leave a comment.

If we now create a `Book`, try to add it to the database, and save it, then you will get an exception if the Author's foreign key is not nullable in the database. Usually that's the right approach, but the InMemory database allows null foreign keys. So if you tried changing the example above and running it, this is why you would not have gotten an exception.

One final note, now that we have removed our ability to set `Book.Author` directly, we are relying on EFCore to do so, which will happen **after** saving to the database. Before that, the author will be null. This is clearly a tradeoff. But if we want to control how a relationship can be created, we should do that from just one side of the relationship and keep ourselves from creating the relationship through an unintended path. 