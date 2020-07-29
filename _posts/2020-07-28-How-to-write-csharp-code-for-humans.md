---
title: "How to write c# code for humans"
toc: true
toc_sticky: true
categories:
  - Programming
tags:
  - C#
---

In this blog post I will show you how to deal with the code smells that leads to significant loss of a tremendous amount of time during the development. Writing clean code should meet a certain rules . Before listing theses rules, I want to start with this quote:

> Any fool can write code that a computer can understand. Good programmers write code that humans can understand.
> <cite><a href="https://en.wikiquote.org/wiki/Martin_Fowler">Martin Fowler</a></cite>

This quote resumes the meaning of a clean code.  
So after solving a problem, you should ask yourself if your code is readable and maintainable as possible.  
To answer to this question,
Check if your code meets the following rules.

# Poor Names

Poor names are the most common code smells that the developers suffered from and they come with different categories.

## Mysterious Names

In this case, the developer cannot figure out the meaning of the variables. He should look somewhere else to understand what these variables reflect.

``` csharp
// Bad Code
var p = new List<decimal> { 25.5m, 50m };
decimal t = 0;
foreach (var i in p)
{
    t += i;
}
return t;
```

``` csharp
// Clean Code
var prices = new List<decimal> { 25.5m, 50m };
decimal total = 0;
foreach (var price in prices)
{
    total += price;
}
return total;
```

**Rule 1:** You must choose a descriptive names so that the developer should not look somewhere else to understand what happens.
{: .notice--success}               