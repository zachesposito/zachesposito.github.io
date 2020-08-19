---
layout: post
title:  "Examples of race conditions in unit tests"
date:   2020-08-18
categories: [technical]
tags: [c#]
---
Recently when optimizing the continuous integration process for a .NET web API, I noticed that a handful of unit tests would occasionally fail, even though no code had changed. There turned out to be two root causes:

## Thread-unsafe collection manipulation
The first group of failing tests all resulted in: `System.InvalidOperationException : Sequence contains more than one matching element`, thrown when calling `searchers.SingleOrDefault` in this static class:

{% highlight csharp %}
public static class SearcherCache
{
  private static readonly ICollection<ISearcher> searchers;
 
  static SearcherCache()
  {
    searchers = new List<ISearcher>();
  }
 
  public static Dictionary<string, string> GetSearcher<T>() where T :ISearcher, new()
  {
    return  GetSearcherIfExists<T>().PropertyMappings;
  }
 
  private static ISearcher GetSearcherIfExists<T>() where T : ISearcher, new()
  {
    var searcher = searchers.SingleOrDefault(t => t.GetType() == typeof(T));
 
    if (searcher == null)
    {
      searcher = new T();
      searchers.Add(searcher);
    }
 
    return searcher;
  }
}
{% endhighlight %}

The intent of this class seemed to be to act as a singleton that lazy-loads `ISearchers`. It appears that only 1 instance is required per implementation of `ISearcher`, so the `GetSearcherIfExists` method attempts to retrieve the instance if it already exists in the `searchers` collection, or add and return a new instance if it does not.

Although that seems like a straightforward way to enforce uniqueness in a collection, this approach breaks down in a concurrent context like a web API. Since multiple requests can be handled by the API at the same time, there is a chance that two or more requests could cause the first few lines in `GetSearcherIfExists` to be executed at the same time resulting in a flow of events like this:

1. Application starts, `searchers` is empty
2. Multiple simultaneous requests cause multiple threads to call `GetSearcherIfExists` with the same value for `T`
3. They simultaneously call `searchers.SingleOrDefault`, setting `searcher` to `null` because `searchers` contains no elements yet
4. Because `searcher` is `null` for all threads, they all enter the `if` block, instantiate `T`, and add the instance to `searchers`
5. The next time `GetSearcherIfExists` is called for that value of `T`, `.SingleOrDefault` throws an exception because `searchers` contains multiple elements of type `T`

The above steps illustrate how the exception occurs in a web environment, but why was the exception occurring when running unit tests? It turns out that the unit tests in the API are made with xUnit, and as of version 2 [xUnit runs separate test classes in parallel by default](https://xunit.net/docs/running-tests-in-parallel.html). There are two test classes in the API that ultimately call `GetSearcher` with the same value of `T`, and since the classes in those tests are run in parallel they can sometimes result in the same flow of events listed above.

Fortunately, an easy way to fix this concurrency bug is to switch the collection type used in FilterSearcher to a [thread-safe collection type](https://docs.microsoft.com/en-us/dotnet/standard/collections/thread-safe/), such as `ConcurrentDictionary`:

{% highlight csharp %}
public static class SearcherCache
{
  private static readonly ConcurrentDictionary<Type, ISearcher> searchers;
 
  static SearcherCache()
  {
    searchers = new ConcurrentDictionary<Type, ISearcher>();
  }
 
  public static Dictionary<string, string> GetSearcher<T>() where T : ISearcher, new()
  {
    return GetSearcherIfExists<T>().PropertyMappings;
  }
 
  private static ISearcher GetSearcherIfExists<T>() where T : ISearcher, new()
  {
    return searchers.GetOrAdd(typeof(T), new T());
  }
}
{% endhighlight %}

`ConcurrentDictionary.GetOrAdd` performs the same conditional-add-and-return functionality intended by `GetSearcherIfExists` previously, but in a way that accounts for the possibility of multiple threads accessing the collection at the same time, whether in a web context or when running unit tests in parallel.

The other root cause of failed unit tests turned out to be simpler, but still surprising:

## DateTimeOffset imprecision

The other group of failing tests were all caused by tests that essentially looked like this (pseudo-code):

{% highlight csharp %}
/*
* Unit test class
*/
[Fact]
public void DateTest(){
  var result = SomeService.GetModelWithDate();
 
  Assert.NotNull(result.Date);
  Assert.True(result.Date < DateTimeOffset.Now);
}
 
//...//
 
/*
* SomeService.cs
*/
public SomeModel GetModelWithDate(){
  return new Model {
    Date = DateTimeOffset.Now
  };
}
{% endhighlight %}

The second assertion would sometimes fail, complaining that `result.Date` was not less than, but actually equal to `DateTimeOffset.Now`! How could that be, since `Date` was clearly set on the model instance before that assertion was run? Those two invocations of `DateTimeOffset.Now` clearly happened at different times!

The problem is `DateTimeOffset` (and `DateTime`) are not as precise as they might seem. [The MSDN docs clarify that these classes depend on the system clock](https://docs.microsoft.com/en-us/dotnet/api/system.datetimeoffset.now?view=netcore-3.1#remarks), and realistically can only discern intervals of several milliseconds or more (the docs offer 10-15 ms as an example). As a result, if `DateTimeOffset.Now` runs twice within several milliseconds, it can return the same value. That's why in the example unit test above, `result.Date` and `DateTimeOffset.Now` are equal at the point of the second assertion.

To fix, I simply removed the second assertion because it didn't seem that it was valuable, but if it turns out it is valuable to check that time difference, the service under test could be refactored to accept a time provider instead, which would allow for more control over unit testing specific timing.