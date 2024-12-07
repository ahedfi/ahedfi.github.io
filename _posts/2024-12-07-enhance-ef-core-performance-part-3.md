---
title: "Enhance EF Core Performance (Part 3)"
permalink : "/Enhance-ef-core-performance-part-3"
header: 
  image : "/images/enhance-ef-core-performance/ef-core-logo.PNG"
  teaser : "/images/enhance-ef-core-performance/ef-core-logo.PNG"
toc: true
toc_sticky: true
categories:
  - Database
tags:
  - EF Core
  - Sql Server
---

Hello again! As we continue our journey into optimizing EF Core, this section builds on the strategies we’ve already explored, such as query projection, no-tracking queries, and batching. But what happens when you need to use custom methods in a LINQ query methods that EF Core doesn’t know how to translate into SQL?

In this part, we’ll dive into this challenge and explore effective strategies to ensure your queries remain performant while leveraging the flexibility of custom logic.

[You can find the full source code used in this post here](https://github.com/ahedfi/EnhanceEfCorePerformance/tree/main/EnhanceEfCorePerformance/CustomMethods)
{: .notice--success} 

# Use Case

Consider a scenario where you have a custom method to perform case-insensitive string matching using .`IndexOf()`:

``` csharp
bool ContainsKeyword(string input, string keyword)
{
    return input != null && input.IndexOf(keyword, StringComparison.OrdinalIgnoreCase) >= 0;
}
```

You might want to use this method inside an EF Core LINQ query, like this:
``` csharp
var result = dbContext.Blogs
    .Where(blog => ContainsKeyword(blog.Name, "EF Core"))
    .ToList();

```

However, EF Core throws an exception because it cannot translate the custom ContainsKeyword method into SQL.

![custom method runtime exception](/images/enhance-ef-core-performance/runtime-exception.jpg)

In the following sections, we will explore how to implement this solution without running into runtime exceptions.

# Client-Side Evaluation

It is an approach to force EF Core to fetch data into memory by calling `AsEnumerable` or `ToList` and apply the custom method afterward:

``` csharp
var result = dbContext.Blogs
    .AsEnumerable() // Switch to in-memory evaluation.
    .Where(blog => ContainsKeyword(blog.Name, "EF Core"))
    .ToList();
```

This may lead to performance issues for **large datasets** as it loads all records into memory.
{: .notice--warning} 

# Expression tree

An expression tree is a hierarchical representation of a piece of code, where each node in the tree corresponds to a part of the code, such as a variable, method call, or operator.
It is widely used in scenarios where code needs to be dynamically constructed, analyzed, or transformed at runtime.

![expression tree](/images/enhance-ef-core-performance/expression-tree.jpg)

## Implementation

Here’s how you can define and use an expression tree to handle case-insensitive string matching:

``` csharp
using System.Linq.Expressions;

public static Expression<Func<T, bool>> CreateContainsKeywordExpression<T>(string propertyName, string keyword)
{
    // Parameter for lambda: x
    var parameter = Expression.Parameter(typeof(T), "x");

    // Access the property: x.Property
    var property = Expression.Property(parameter, propertyName);

    // Ensure property is not null
    var notNullCheck = Expression.NotEqual(property, Expression.Constant(null, typeof(string)));

    // Use EF.Functions.Like: EF.Functions.Like(x.Property, '%keyword%')
    var likeMethod = typeof(DbFunctionsExtensions).GetMethod(nameof(DbFunctionsExtensions.Like),
        new[] { typeof(DbFunctions), typeof(string), typeof(string) });

    var efFunctions = Expression.Constant(EF.Functions);
    var keywordPattern = Expression.Constant($"%{keyword}%", typeof(string));
    var likeCall = Expression.Call(likeMethod, efFunctions, property, keywordPattern);

    // Combine the null check and Like call: x.Property != null && EF.Functions.Like(x.Property, '%keyword%')
    var finalCondition = Expression.AndAlso(notNullCheck, likeCall);

    // Create the expression tree: x => x.Property != null && EF.Functions.Like(x.Property, '%keyword%')
    return Expression.Lambda<Func<T, bool>>(finalCondition, parameter);
}

```

Here’s how to use the expression tree in your query:

``` csharp
var keyword = "EF CORE";
var expression = QueryExtensions.CreateContainsKeywordExpression<Blog>("Name", keyword);

using var context = new AppDbContext();
var blogs = context.Blogs.Where(expression).ToList();

foreach (var blog in blogs)
{
    Console.WriteLine($"Blog ID: {blog.Id}, Name: {blog.Name}");
}
```

**Generated SQL**

With this approach, EF Core can generate efficient SQL queries like:
``` sql
SELECT [b].[Id], [b].[Author], [b].[CreationDate], [b].[Description], [b].[Name]
    FROM [Blogs] AS [b]
    WHERE [b].[Name] LIKE N'%EF CORE%'
```

The benefits you gain from utilizing expression trees in LINQ queries are the following:

- **Custom Logic in LINQ**: Allows complex custom logic to be integrated seamlessly into EF Core queries.
- **SQL Translation**: Ensures EF Core can translate your logic into SQL, avoiding runtime exceptions.
- **Reusable**: Expression trees can be reused for similar queries across your application.

## Limitations

Writing expression trees can be more complex and less intuitive than standard LINQ.

# FromSqlRaw  

The `FromSqlRaw ` method enables us to write SQL queries that are executed on the database side, bypassing EF Core's query translation limitations. 
Instead of relying on EF Core to translate LINQ expressions into SQL, `FromSqlRaw ` gives us full control over the SQL code, 
enabling us to use custom SQL functions and methods that EF Core might not support natively.

## Implementation 

Here's an example of how to use `FromSqlRaw`:
``` csharp
public List<Blog> FilterBlogsUsingFromSql(string keyword)
{
    // Define the raw SQL query with a parameter placeholder
    string sqlQuery = @"
        SELECT * FROM Blogs
        WHERE Name LIKE '%' + {0} + '%'
    ";

    // Use FromSqlRaw to execute the query with the parameter
    return _dbContext.Blogs
        .FromSqlRaw(sqlQuery, keyword)  // Passing the 'keyword' as a parameter
        .ToList();
}
``` 
For more complex queries, you can call a stored procedure using `FromSqlRaw`. This method allows you to execute a raw SQL query and map the results directly to your entity types.

``` csharp
public List<Blog> GetBlogsByKeyword(string keyword)
{
    var parameter = new SqlParameter("@Keyword", keyword);
    return _dbContext.Blogs.FromSqlRaw("EXEC GetBlogsByKeyword @Keyword", parameter).ToList();
}
```

By this approach, you gain the following advantages:
- **Custom SQL**: You can leverage the full power of SQL, including functions and operations that EF Core cannot translate.
- **No Translation Issues**: By bypassing EF Core's translation engine, you avoid runtime exceptions related to unsupported methods.
- **Performance**: Executing raw SQL queries can sometimes be more <u>efficient</u>, especially for <u>complex operations</u>, as you can optimize the query to meet specific performance requirements.


## Limitations
When returning entity types from SQL queries, keep in mind that:
- Losing the benefits of LINQ, such as compile-time checking
- The query must return all entity properties.
- Column names in the result set must match the mapped property names.
- The query can't include related data, but you can use Include to load related entities.


# References

Below are the references and resources I used to prepare this post:
- [https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)
- [https://learn.microsoft.com/en-us/ef/core/querying/sql-queries](https://learn.microsoft.com/en-us/ef/core/querying/sql-queries)


Stay tuned for the final post in this series, where we will discuss caching and how it can significantly enhance EF Core performance.