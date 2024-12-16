---
title: "Enhance EF Core Performance (Part 4)"
permalink : "/Enhance-ef-core-performance-part-4"
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

In this final post of our series on optimizing EF Core performance, we’ll dive into a powerful performance-boosting technique: caching.

Caching plays a critical role in modern application optimization by temporarily storing frequently accessed data. This reduces the need for repetitive database queries, shortens response times, and lowers the load on the database. The result is an improved user experience and greater scalability, especially under high traffic conditions.

EF Core supports caching through the following mechanisms:


# First-Level Cache

EF Core automatically provides a **first-level cache**, which is scoped to the lifecycle of the DbContext. 

![first-level cache](/images/enhance-ef-core-performance/first-level%20cache.png)

This means when you query for an entity, EF Core first checks the change tracker to see if the entity is already loaded. If it is, EF Core returns the entity from memory instead of querying the database again.

## Implementation

This caching is implicit and requires no additional configuration.

``` csharp
using var context = new AppDbContext();
var blog1 = context.Blogs.FirstOrDefault(b => b.Id == 1); // Queries the database
var blog2 = context.Blogs.FirstOrDefault(b => b.Id == 1); // Returns from the cache
```

## Limitations

Here are some limitations of the First-Level Cache in EF Core:
- It is limited to the lifetime of a single `DbContext` instance. Once the `DbContext` is disposed of, the cached data is no longer available.
- Data cached in one `DbContext` instance is not shared with other `DbContext` instances. This can lead to redundant queries if multiple contexts access the same data.
- By default, EF Core tracks all retrieved entities in the First-Level Cache, which can introduce performance overhead if tracking is unnecessary. Using `AsNoTracking()` disables tracking but also bypasses caching.
- It only applies to entity queries. Raw SQL queries or projections using LINQ to retrieve non-entity will not be cached.

# Second-Level Cache

The **Second-Level Cache** is a caching mechanism that extends beyond the lifecycle of a single `DbContext` instance. It allows for shared caching of query results or entities across multiple `DbContext` instances within an application. 

![second-level cache](/images/enhance-ef-core-performance/second-level%20cache.png)

Using this cache, you can store the results of specific queries or entities based on your application's requirements and configure expiration policies to determine when the cached data should be invalidated or refreshed.

<u>EF Core does not have a built-in second-level cache</u>. However, it can be achieved using third-party libraries.

In this post, I chose to showcase **[EFCore Second-Level Cache Interceptor](https://github.com/VahidN/EFCoreSecondLevelCacheInterceptor)**. This library stands out due to its rich set of features, including:

- Seamless integration with EF Core using interceptors.
- Support for caching query results across different DbContext instances.
- Flexibility to use various cache providers like MemoryCache, Redis, or NCache.
- Advanced configuration options for expiration policies and fine-grained caching control.
- Comprehensive logging for cache hits and misses, making it easy to monitor cache behavior.

## Implementation

Below is a step-by-step guide to demonstrate how to integrate and use **EFCore Second-Level Cache Interceptor**.

**Step 1**

Install the NuGet package:

``` bash
dotnet add package EFCoreSecondLevelCacheInterceptor
```
**Step 2**

Add the required configuration during registering the services.

``` csharp
using EFCoreSecondLevelCacheInterceptor;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
 builder.Services.AddDbContext<BloggingDbContext>((serviceProvider, options) => 
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
           .AddInterceptors(serviceProvider.GetRequiredService<SecondLevelCacheInterceptor>())); // Add SecondLevelCacheInterceptor to DbContextOptionsBuilder pipeline

 builder.Services.AddEFSecondLevelCache(options =>
     options.UseMemoryCacheProvider()  // Use in-memory caching
            .ConfigureLogging(true)    // Enable logging for cache operations
 );

var app = builder.Build();
app.Run();

```
If you don't want use the built-in In-Memory cache provider, you can use the following cache provider:
- [EasyCaching.Core cache provider](https://github.com/VahidN/EFCoreSecondLevelCacheInterceptor/tree/master#using-easycachingcore-as-the-cache-provider)
- [EasyCaching.Core dynamic cache provider](https://github.com/VahidN/EFCoreSecondLevelCacheInterceptor/tree/master#using-easycachingcore-as-a-dynamic-cache-provider)
- [CacheManager.Core cache provider](https://github.com/VahidN/EFCoreSecondLevelCacheInterceptor/tree/master#using-cachemanagercore-as-the-cache-provider-its-not-actively-maintained)
- [A custom cache provider](https://github.com/VahidN/EFCoreSecondLevelCacheInterceptor/tree/master#using-easycachingcore-as-the-cache-provider)

**Step 3**

Now, you can use the .Cacheable() extension method on your EF Core LINQ queries to enable caching.

``` csharp
using EFCoreSecondLevelCacheInterceptor;

public class ProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<List<Product>> GetProductsAsync()
    {
        var products = await _context.Products
                .Where(p => p.Price > 100)
                .OrderBy(p => p.Name)
                .Cacheable() // Enable caching for this query
                .ToListAsync();

        return products;
    }
}
```

You can configure cache expiration policies and fine-tune caching behavior:

``` csharp
public async Task<List<Product>> GetProductsAsync()
    {
        var products = await _context.Products
            .Where(p => p.Price > 100)
            .OrderBy(p => p.Name)
            .Cacheable(CacheExpirationMode.Sliding, TimeSpan.FromMinutes(5)) // Enable caching for this query
            .ToListAsync();

        return products;
    }
```

- `CacheExpirationMode.Absolute`: Sets an absolute expiration time.
- `CacheExpirationMode.Sliding`: Resets the expiration timer whenever the cache is accessed.

**Step 4**

To verify that caching is working, enable logging to see cache hits/misses. The logs will show messages like:

``` bash
dbug: EFCoreSecondLevelCacheInterceptor.DbCommandInterceptorProcessor[100001]
      TableRows[ad57b5fc-7b27-4a04-9506-5df179266963] added to the cache[KeyHash: EF_CCD64A079264F5E1, DbContext: BloggingDbContext, CacheDependencies: EF_Blogs.].
dbug: EFCoreSecondLevelCacheInterceptor.EFCacheDependenciesProcessor[0]
      ContextTableNames: Blogs, PossibleQueryTableNames: Blogs -> CacheDependencies: Blogs.
dbug: EFCoreSecondLevelCacheInterceptor.EFCacheKeyProvider[0]
      KeyHash: EF_CCD64A079264F5E1, DbContext: BloggingDbContext, CacheDependencies: EF_Blogs.
dbug: EFCoreSecondLevelCacheInterceptor.DbCommandInterceptorProcessor[0]
      Suppressed the result with the TableRows[ad57b5fc-7b27-4a04-9506-5df179266963] from the cache[KeyHash: EF_CCD64A079264F5E1, DbContext: BloggingDbContext, CacheDependencies: EF_Blogs.].
dbug: EFCoreSecondLevelCacheInterceptor.EFCachePolicyParser[0]
      Using EFCachePolicy: EFCachePolicy --> Absolute|00:30:00|||False.
dbug: EFCoreSecondLevelCacheInterceptor.DbCommandInterceptorProcessor[100000]
      Returning the cached TableRows[ad57b5fc-7b27-4a04-9506-5df179266963].
```

[To investigate more, you can find an example here](https://github.com/ahedfi/EnhanceEfCorePerformance/tree/main/EnhanceEfCorePerformance/Caching)
{: .notice--success} 

## Limitations

While second-level caching in EF Core can significantly improve performance, there are several limitations and considerations to keep in mind:
- Cached data might become outdated if the underlying database is updated outside of the application (e.g., direct SQL changes, another application modifying data).
- Queries with projections (e.g., Select) might not always be cacheable.
- Keeping cache and database in sync often requires additional mechanisms like cache invalidation, which adds complexity.
- Second-level caching stores data in memory or an external caching provider. Improper cache eviction policies or a high volume of data can lead to excessive memory usage, potentially degrading overall application performance.


This brings us to the end of our series on enhancing EF Core performance. I hope you’ve found these tips and techniques useful for improving your application's efficiency and scalability.
While this series is over, I’ll continue to focus on EF Core in future posts. Stay tuned as I dive into another powerful feature **interceptors** and explore how they can elevate your EF Core experience!