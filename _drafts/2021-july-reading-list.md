---
layout: single
classes: wide
title:  "2021 July Reading List"
date:   2021-08-05 09:00:00 -0400
permalink: blog/2021/2021-july-reading-list
---

Taking a bit of inspiration from the [Jo Walton's Reading List series](https://www.tor.com/tag/jo-walton-reads/)

### Maarten Balliauw - [Building a supply chain attack with .NET, NuGet, DNS, source generators, and more!](https://blog.maartenballiauw.be/post/2021/05/05/building-a-supply-chain-attack-with-dotnet-nuget-dns-source-generators-and-more.html)

Discussed several surprising things that could be done in supply chain attacks. In particular he showed several ways that malicious code could be hidden from users, by masking it from IntelliSense and debugging. 

The most interesting point was what you could with Source Generators. Normally, visual studio makes it easy to inspect the generated code. But he pointed out that the generator could attempt detect when a CI build is running and only in that case would it add malicious code to your output. In that situation, the dev team may never see the code is being added.

Incidentally, his blog gave me a push to move my blog onto Jekyll. 

### Alex Birsan - [Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies The Story of a Novel Supply Chain Attack](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)

Demonstrates how private dependencies may be hijacked by creating a public library with an identical name. Provides an interesting look into hacking for bug bounties. 

### Microsoft - [3 ways to mitigate risk when using private package feeds](https://aka.ms/pkg-sec-wp) Whitepaper

Gives some details on avoiding the attack in the previous article. 

### Amadeusz Sadowski - [List of C# Source Generators](https://github.com/amis92/csharp-source-generators)

A list of C# source generators, mostly available as NuGet packages. Also has a few projects that use generators internally.

### Mark Seemann - [Property-based testing is not the same as partition testing](https://blog.ploeh.dk/2021/06/28/property-based-testing-is-not-the-same-as-partition-testing/)

I hadn't heard of partition testing before. Though its interesting in that a lot of my tdd process revolves around partitioning into examples. I haven't done any true property based testing (true here meaning randomly generating the test data) and really should look into FsCheck more, which Mark reminded me can be used from C# [even though it works better in F#](https://blog.ploeh.dk/2021/02/15/when-properties-are-easier-than-examples/).