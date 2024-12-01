---
title: "Enhance EF Core Performance (Part 2)"
permalink : "/Enhance-ef-core-performance-part-2"
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

Welcome back to Enhance EF Core Performance series! 🚀 In Part 1, we laid a solid foundation by exploring techniques like **DbContext pooling**, **compiled queries**, and **split queries**.

In this second part, we’re shifting focus to practical query optimization techniques.
These strategies will help you fine-tune your EF Core queries for real-world scenarios, ensuring your application stays responsive and efficient under heavy loads.

Let’s dive in and unlock the next level of performance optimization!

# Query Projection

It is a fundamental technique to improve EF Core performance by retrieving only the data you actually need. Instead of fetching entire entities from the database, you project the query results into simpler objects or anonymous types that contain only the required fields. This reduces the amount of data transferred from the database and minimizes memory usage, leading to faster queries.

## Implementation

Imagine a `Blog` entity with several related fields:
``` csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Author { get; set; }
    public DateTime CreationDate { get; set; }
    public string Description { get; set; }
}
```
If you want only the blog name and author, but you query the entire entity: 
``` csharp
var blogs = await _context.Blogs.ToListAsync();
``` 
This fetches all fields, including `Description` and `CreationDate`, even if they aren’t needed. This approach wastes bandwidth and memory.

You can optimize the query by using **projection** to select only the fields you need:
``` csharp
var blogs = await _context.Blogs
    .Select(b => new { b.Name, b.Author })
    .ToListAsync();
``` 
This fetches only the `Name` and `Author` fields, significantly reducing the data transferred and processed.

Instead of anonymous types, you can project the query into **Data Transfer Objects (DTOs)** for better type safety and reuse:
``` csharp
public class BlogDto
{
    public string Name { get; set; }
    public string Author { get; set; }
}

var blogs = await _context.Blogs
    .Select(b => new BlogDto
    {
        Name = b.Name,
        Author = b.Author
    })
    .ToListAsync();
``` 
This approach is especially useful when returning data from an API or handling complex business logic.


## Benchmarks
Here’s a comparison of performance between querying the entire entity versus projecting specific fields:

![projection query benchmark](/images/enhance-ef-core-performance/projection.jpg)

- **Execution Time**: Fetching only required fields (With Projection) is significantly faster.
- **Memory Usage**: Projected queries allocate far less memory compared to fetching full entities.

[You can find the full source code for this benchmark here](https://github.com/ahedfi/EnhanceEfCorePerformance/tree/main/EnhanceEfCorePerformance/Projection)
{: .notice--success} 

## Limitations

- **No Lazy Loading**: When projecting, navigation properties are not loaded unless explicitly included.
- **Complex Projections**: Projections with complex logic might affect query readability and maintainability.

# Pagination

It is a technique used to retrieve data in smaller chunks rather than loading the entire dataset into memory. In EF Core, pagination is implemented using the `Skip` and `Take` methods, which enable you to limit the number of records fetched from the database. This approach is particularly beneficial for the following cases:
- **Improved Performance**: Fetching only the required records reduces query execution time and network overhead.
- **Scalability**: By handling smaller chunks of data, your application remains responsive even as the dataset grows.
- **User Experience**: Pagination is commonly used in UI components such as tables or lists to provide a smoother user experience.

## Implementation 
Here's an example of how to implement pagination:
``` csharp
public async Task<List<BlogDto>> GetBlogDtosAsync(int pageNumber, int pageSize)  
{  
    return await dbContext.Blogs  
        .OrderBy(b => b.Id)  
        .Skip((pageNumber - 1) * pageSize)  
        .Take(pageSize)  
        .Select(b => new BlogDto { Id = b.Id, Title = b.Title })  
        .ToListAsync();  
}  
``` 
- `OrderBy`: Ensures a consistent ordering of records.
- `Skip`: Skips the records from previous pages.
- `Take`: Fetches only the required number of records for the current page.

## Limitations

- **Consistency**: Without a stable sort order (OrderBy), results may vary between queries.
- **Performance on Large Offsets**: Using Skip with large offsets can degrade performance as the dataset grows.

# No-Tracking Queries

By default, EF Core tracks every entity it retrieves from the database in its change tracker, which enables it to detect modifications and persist changes back to the database. However, this tracking comes with overhead that can be avoided for **read-only operations**.

The following query tracks the retrieved entities:
``` csharp
var blogs = await _context.Blogs.ToListAsync();
``` 
This means EF Core keeps track of the Blog entities in the ChangeTracker, even if you don’t modify them. This tracking consumes **additional memory and CPU cycles**, especially for large datasets.

**No-tracking queries** are ideal for the following cases:

- **Read-Only Data**: When the data will not be modified in the current context.
- **High-Performance Reads**: When querying a large dataset or frequently accessed data.
- **API Endpoints**: When serving read-only requests in RESTful APIs such as returning data for GET endpoints.

## Implementation
By using the `.AsNoTracking()` method, you can disable tracking for a specific query:
``` csharp
var blogs = await _context.Blogs
    .AsNoTracking()
    .ToListAsync();
``` 
In this case, EF Core retrieves the Blog entities without tracking them, reducing overhead.

If your application primarily performs read-only operations, you can configure EF Core to use no-tracking queries globally for an entire `DbContext`:
``` csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}
```
## Benchmarks

Here’s a comparison of the performance impact:

![no tracking query benchmark](/images/enhance-ef-core-performance/no-tracking.jpg)

- **Execution Time**: No-tracking queries (With No-Tracking) are faster because they skip the overhead of the change tracker.
- **Memory Usage**: No-tracking queries allocate less memory since EF Core doesn’t track entity states.

[You can find the full source code for this benchmark here](https://github.com/ahedfi/EnhanceEfCorePerformance/tree/main/EnhanceEfCorePerformance/NoTracking)
{: .notice--success} 

## Limitations

- Change Detection Disabled: You cannot modify the entities retrieved with no-tracking queries in the same context.
- Explicit Tracking Needed: If you later decide to track an entity from a no-tracking query, you’ll need to attach it manually:
``` csharp
_context.Attach(entity);
```
- Use `AsNoTrackingWithIdentityResolution`: Use it when `AsNoTracking` alone leads to redundant or incorrect object instances due to lack of identity resolution.

# Loading Related Data

Loading related data is a common task in Entity Framework Core (EF Core) when you need to fetch data from multiple related entities. EF Core provides several ways to load related data. Each method has its own performance implications and use cases, so it's important to choose the right strategy depending on the scenario.

## Implementation

EF Core provides three strategies for loading related data.

### Eager Loading

Eager Loading retrieves the related data as part of the initial query using the `Include` method. This method is useful when you know you will need the related data immediately and want to load everything in one go, avoiding the "N+1" query problem.
``` csharp
public async Task<List<Blog>> GetBlogsWithPostsAsync()  
{  
    return await dbContext.Blogs  
        .Include(b => b.Posts)  
        .ToListAsync();  
}  
```

### Lazy Loading

Lazy Loading delays the loading of related data until it is specifically accessed. EF Core supports Lazy Loading when you configure the navigation properties with virtual keyword and use proxies. This approach avoids fetching related data unless it is needed, but it can result in additional database queries during runtime.
``` csharp
public class Blog  
{  
    public int Id { get; set; }  
    public string Title { get; set; }  
    public virtual ICollection<Post> Posts { get; set; } // Virtual for lazy loading  
}
```

Lazy loading can result in multiple database queries, leading to potential performance issues if overused, especially in loops.
{: .notice--warning} 

### Explicit Loading

Explicit Loading allows you to load related data only when you explicitly request it. This gives you more control compared to Lazy Loading, and it's better suited when you want to load related data after the main entity has already been retrieved.
``` csharp
public async Task LoadPostsExplicitlyAsync(int blogId)  
{  
    var blog = await dbContext.Blogs  
        .FirstOrDefaultAsync(b => b.Id == blogId);  
    
    if (blog != null)  
    {  
        await dbContext.Entry(blog)  
            .Collection(b => b.Posts)  
            .LoadAsync();  
    }  
}  
```

Choosing the right data-loading strategy in Entity Framework Core is essential for optimizing performance and meeting your application's requirements. The table below highlights the three main approaches to loading related data:

| Loading Type    | When to Use                                | Performance Impact                              |
|------------------|-------------------------------------------|------------------------------------------------|
| Eager Loading    | When related data is needed right away.   | Single query, may lead to heavy joins.         |
| Lazy Loading     | When related data is accessed intermittently. | Multiple queries, risk of N+1 problem.       |
| Explicit Loading | When you need to load related data conditionally. | Control over when data is loaded, less risk of excessive queries. |

## Limitations

- **Use Eager Loading Judiciously**: Eager loading can generate complex and resource-intensive SQL queries, potentially affecting performance. Opt for it only when the related data is essential.
- **Avoid Lazy Loading in Loops**: Lazy loading within loops can trigger multiple database calls, significantly impacting performance. Use it cautiously to prevent unnecessary overhead.
- **Leverage Explicit Loading for Conditional Needs**: Explicit loading provides better control, making it ideal for scenarios where related data is required only under specific conditions, avoiding the drawbacks of lazy loading.

# Batching

Batching in Entity Framework Core (EF Core) is an optimization technique that combines multiple database operations into a single round trip to the database server. 
This reduces network overhead and enhances performance, particularly in data-intensive applications. 
When methods like `SaveChanges` or `SaveChangesAsync` are invoked, EF Core groups operations (such as inserts, updates, and deletes) 
into batches and sends each batch as a single command to the database. For instance, if you add several entities to the DbContext and call `SaveChanges`, 
EF Core will batch the corresponding `INSERT` statements. Similarly, updates or deletions of multiple entities are grouped into `UPDATE` or `DELETE` statement batches.

## Implementation

You can control the size of batches using the `MaxBatchSize` option in the `DbContext` configuration. The default batch size is provider-specific, but you can optimize it for your workload.
``` csharp
public class AppDbContext : DbContext  
{  
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)  
    {  
        optionsBuilder.UseSqlServer("YourConnectionString", sqlOptions =>  
        {  
            sqlOptions.MaxBatchSize(50); // Customize batch size  
        });  
    }  
}    
```
Starting from EF Core 7.0, the `ExecuteUpdate` and `ExecuteDelete` methods allow you to perform **bulk updates** and **bulk deletes** directly on the database without loading the entities into memory. 
These methods are designed for scenarios where you need to modify or delete multiple rows at once, making them highly efficient for such operations.

**Example**
``` csharp
await dbContext.Blogs  
    .Where(b => b.Tag == "Tech")  
    .ExecuteUpdateAsync(update => update.SetProperty(b => b.IsFeatured, true));

await dbContext.Posts  
    .Where(p => p.CreatedAt < DateTime.UtcNow.AddYears(-1))  
    .ExecuteDeleteAsync();
```

**Generated SQL**
``` sql
UPDATE Blogs  
SET IsFeatured = 1  
WHERE Tag = 'Tech';  

DELETE FROM Posts  
WHERE CreatedAt < '2023-01-01';
```

## Limitations

While batching can provide performance improvements in specific scenarios, it also has limitations that should be taken into account.
- **Transaction Size**: Batching a large number of commands can lead to large transactions, which might increase lock contention on the database.
- **Batch Size Configuration**: An inappropriate batch size can either overwhelm the database (if too large) or lead to inefficiency (if too small).
- **Provider Support**: Some database providers may not fully support batching. Verify that your database supports it before relying on batching for performance gains.


# References

Below are the references and resources I used to prepare this post:
- [https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying)
- [https://learn.microsoft.com/en-us/ef/core/performance/efficient-updating](https://learn.microsoft.com/en-us/ef/core/performance/efficient-updating)


Stay tuned for upcoming posts in this series, as we continue to explore and dive deeper into each technique and best practice.