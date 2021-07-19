---
layout: single
classes: wide
title:  "Snippets to Fix FxCop CA1062"
date:   2020-06-08 09:00:00 -0400
permalink: blog/snippets-to-fix-fxcop-ca1062
---

If you run FxCop you may frequently run into [CA1062: Validate arguments of public methods](https://docs.microsoft.com/en-us/visualstudio/code-quality/ca1062). Below are three snippets to make writing this boiler plate a bit easier. 

Before we get to the snippets, if you are in a position to disable `CA1062` and still want thorough null checks, you might consider using [IL weaving with Fody](https://github.com/Fody/NullGuard) to automate the addition of null checks.

### Example

Here is a quick example. This public method uses `input` before validating that it is not null. This results in the error shown below. 

``` csharp
public class FxCopTest
{
    public static void MissingValidation(string input)
    {
        if (input.Length != 0)
        {
            Console.WriteLine(input);
        }
    }
}
```

![](/images/2020/snippets-to-fix-fxcop-ca1062/ca1062.png)

### Snippets

The snippets below all provide the same functionality, but differ in their style.

To install the snippet in Visual Studio. 
1. Press Ctrl+K, Ctrl+B to open the Code Snippet Manager, where you can get the path to the `Visual C#\My Code Snippets`. 
2. Save the selected snippet with any name, and the `.snippet` file extension. 
3. You may need to restart Visual Studio for the snippet to become available.

![](/images/2020/snippets-to-fix-fxcop-ca1062/codesnippetmanager.png)

Usage for each snippet is the same:
1. Type `nullguard` or enough of the name that IntelliSense selects `nullguard`.
2. Hit tab twice.
3. Type the name of the argument to check and press enter/return.

Each option below shows example output from the snippet, along with the snippet itself.

### Default Option

*Note, this snippet is not safe to use with StyleCop. See the next option.*

``` csharp
if (input == null) throw new ArgumentNullException(nameof(input));
```
``` xml
<?xml version="1.0" encoding="utf-8" ?>
<CodeSnippets  xmlns="http://schemas.microsoft.com/VisualStudio/2005/CodeSnippet">
    <CodeSnippet Format="1.0.0">
        <Header>
            <Title>nullguard</Title>
            <Shortcut>nullguard</Shortcut>
            <Description>Code snippet for performing null check.</Description>
            <Author>Mike Conrad - https://www.mjconrad.com/</Author>
            <SnippetTypes>
                <SnippetType>Expansion</SnippetType>
            </SnippetTypes>
        </Header>
        <Snippet>
            <Declarations>
                <Literal>
                    <ID>argument</ID>
                    <ToolTip>Argument to check for null</ToolTip>
                    <Default>argument</Default>
                </Literal>
            </Declarations>
            <Code Language="csharp"><![CDATA[if ($argument$ == null) throw new ArgumentNullException(nameof($argument$));]]>
            </Code>
        </Snippet>
    </CodeSnippet>
</CodeSnippets>
``` 

### StyleCop Safe Option

This snippet avoids avoids violating these two StyleCop rules:

* SA1503 - Braces should not be omitted
* SA1501 - Statement should not be on a single line

``` csharp
input = input ?? throw new ArgumentNullException(nameof(input));
```

``` xml
<?xml version="1.0" encoding="utf-8" ?>
<CodeSnippets  xmlns="http://schemas.microsoft.com/VisualStudio/2005/CodeSnippet">
    <CodeSnippet Format="1.0.0">
        <Header>
            <Title>nullguard</Title>
            <Shortcut>nullguard</Shortcut>
            <Description>Code snippet for performing null check.</Description>
            <Author>Mike Conrad - https://www.mjconrad.com/</Author>
            <SnippetTypes>
                <SnippetType>Expansion</SnippetType>
            </SnippetTypes>
        </Header>
        <Snippet>
            <Declarations>
                <Literal>
                    <ID>argument</ID>
                    <ToolTip>Argument to check for null</ToolTip>
                    <Default>argument</Default>
                </Literal>
            </Declarations>
            <Code Language="csharp"><![CDATA[$argument$ = $argument$ ?? throw new ArgumentNullException(nameof($argument$));]]>
            </Code>
        </Snippet>
    </CodeSnippet>
</CodeSnippets>
```

### Helper Class Option

The last option uses a helper class

``` csharp
NullGuard.Check(things, nameof(things));
```

``` csharp
public static class NullGuard
{
    /// <summary>
    /// Throws an <see cref="ArgumentNullException"/> if the provided argument is null.
    /// </summary>
    /// <param name="argument">Argument to check.</param>
    /// <param name="name">Original name of the argument.</param>
    public static void Check(object argument, string name)
    {
        if (argument == null)
        {
            throw new ArgumentNullException(name);
        }
    }
}
```

``` xml

<?xml version="1.0" encoding="utf-8" ?>
<CodeSnippets  xmlns="http://schemas.microsoft.com/VisualStudio/2005/CodeSnippet">
    <CodeSnippet Format="1.0.0">
        <Header>
            <Title>nullguard</Title>
            <Shortcut>nullguard</Shortcut>
            <Description>Code snippet for performing null check.</Description>
            <Author>Mike Conrad - https://www.mjconrad.com/</Author>
            <SnippetTypes>
                <SnippetType>Expansion</SnippetType>
            </SnippetTypes>
        </Header>
        <Snippet>
            <Declarations>
                <Literal>
                    <ID>argument</ID>
                    <ToolTip>Argument to check for null</ToolTip>
                    <Default>argument</Default>
                </Literal>
            </Declarations>
            <Code Language="csharp"><![CDATA[if ($argument$ == null) throw new ArgumentNullException(nameof($argument$));]]>
            </Code>
        </Snippet>
    </CodeSnippet>
</CodeSnippets>
```
