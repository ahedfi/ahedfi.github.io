---
title: "C# code for humans"
permalink : "/Csharp-code-for-humans/"
header: 
  image : "/images/clean-code.jpg"
  teaser : "/images/clean-code.jpg"
toc: true
toc_sticky: true
categories:
  - Programming
tags:
  - C#
---

In this blog post, I will show you how to deal with the code smells that lead to significant loss of a tremendous amount of time during the development.  
Writing clean code should meet certain rules. Before listing these rules, I want to start with this quote:

> Any fool can write code that a computer can understand. Good programmers write code that humans can understand.
> <cite><a href="https://en.wikiquote.org/wiki/Martin_Fowler">Martin Fowler</a></cite>

This quote summarizes what I am intending to explain.    
So after solving a problem, you should ask yourself if your code is readable and maintainable as possible.  
To answer this question, check if your code meets the following rules.

# Poor Names

Poor names are the most common code smells that the developers suffered from and they come with different categories.

## Mysterious Names

Some developers use a short name to write less code. The result, they become the only ones who know the meaning of these names.

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

**Rule 1:** Choose descriptive names that reveal your intentions.
{: .notice--success}          

## Meaningless names

Another category of poor names is meaningless names. Let's take a look at the name of this method.

``` csharp
// Bad Code
void ValidateMethod_SetItemsServerSideToDataStoreCollection() { 
  // code ...
}
```
In this case, the developer should look at the implementation to understand what this method is doing.  
Usually, the name of a method like that is caused by the fact that the method has many things to do, and picking a clean intention name is not easy.  
To avoid this, keep your method short (let's say 10 lines). Then, it will do one thing and you can pick a meaningful name.


**Rule 2:** Choose meaningful names so that the developer should not look somewhere else to understand the code.
{: .notice--success}    


## Hungarian notation

It was very popular to prefix the variable's name with the data type. Nowadays, it is useless to do that because the IDE makes determining types very easy via tooltips.

``` csharp
// Bad Code
int iMaxSize;
string strFirstName;
```

``` csharp
// Clean Code
int maxSize;
string firstName;
```
**Rule 3:** Avoid Hungarian notation or any other type identification in the variable's name.
{: .notice--success}

## Noisy Names


We mean by noisy names those contain unnecessary words. A good name should contain only the essential words to express the concept.

``` csharp
// Bad Code
Contact theContact;
IEnumerable<Contact> listOfContacts;
```

``` csharp
// Clean Code
Contact contact;
IEnumerable<Contact> contacts;
```

**Rule 4:** Avoid Disinformation.
{: .notice--success}


# .NET Naming Conventions

In .NET, there are two naming conventions:

* <span style="color:orange"> PascalCase </span> : the first letter of each word is uppercase
* <span style="color:purple"> camelCase </span> : the first letter of the first word is lowercase but the first letter of each word after that is uppercase

``` csharp
public class Contact 
{
   public static const string ProspectType = "Prospect";

   private int _id;
   protected string Status = string.empty;
   public string Name {get; set;}

   public bool Validate(string name)
   {
      var isValid = false; 
      // code 
      // .
      // .
      // .
      return isValid;
   }
}
```
**Rule 5:** Use **pascal case** for the **class name**, **public properties** and **method name** . For the **private fields**, **method arguments** and **local variables** use **camel case**.
{: .notice--success}

**N.B:** prefix the private field with underscore and 
use pascal case to the fields when its access modifier is different than private.  
{: .notice--primary}

## Naming Classes

The class name should be chosen from problem domain so that the next time the another developer can easily understand and use the code.


You should follow specific guidelines for naming classes:
* Good class names are nouns not verbs. 
* The name should be specific to create a [cohesive class](https://en.wikipedia.org/wiki/Cohesion_(computer_science))
* The noun should lead the design to the single responsablity

``` csharp
// Bad Code
Common
Utility
MyEntity
```

``` csharp
// Clean Code
Contact
Account
ActivityRepository
```


**Rule 6:** Use meaningful names in domain context
{: .notice--success}

## Naming Methods

As we mentioned before, we should avoid meaningless names. A good method name should be meaningful without looking at the implementation.  

If you find yourself using one of these words: `And`,`Or`, `If` and `_`,  you probably need to split the method because it is a sign that it has a lot of things to do.


``` csharp
// Bad Code
Get
Process
ValidateAndLogin
```

``` csharp
// Clean Code
GetUser
SendNotification
Validate
Login
```

**Rule 7:** The developer should not look at the implementation to understand what the method does
{: .notice--success}

## Naming boolean variables

``` csharp
// Bad Code
status
login
close
if(status)
{

}
```

``` csharp
// Clean Code
isEnabled
isLogged
isClosed
if(isEnabled)
{

}
```

**Rule 8:** Boolean names should sound like asking true/false questions
{: .notice--success}
