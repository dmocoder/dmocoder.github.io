---
layout: post
title:  "C# and JSON-UTF8"
date:   2020-02-09 21:35:00 +0000
categories: csharp json utf8
---
JSON (JavaScript Object Notation) is a great data format. It is human readable and widely supported. Converting C# classes to and from Json however can impair performance.

#### Why?
Json is typically encoded in UTF-8 but .NET uses [UTF16 encoding to represent characters and strings](https://docs.microsoft.com/en-us/dotnet/standard/base-types/character-encoding). 
This means the following method will first allocate a UTF16 string before encoding it into UTF8:

{% highlight csharp %}
using Newtonsoft.Json;

public static byte[] SerializeToUtf8(object myObject)
{
    var jsonText = JsonConvert.SerializeObject(myObject);

    return Encoding.UTF8.GetBytes(jsonText);
}
{% endhighlight %}


Allocating an extra string like this isn't the end of the world. The string falls out of scope after the method returns and the memory allocated will be returned by a Garbage Collector (GC) compaction. 

However in a very high throughput application a GC compaction could have quite a notable impact on processing (more on this in a later blog). Less allocations can ultimately mean less pressure on the GC and thus less frequent compactions. TL; DR; Less allocations good.

_(Quick disclaimer: Whilst reducing memory allocations is always a positive thing, often the code to facilitate such can be quite obtuse. It is always recommended to measure and profile performance before you start with micro-optimizations)._


___


This post, and future posts, will be exploring how we can analyze and improve the performance of JSON Serialization.

In my experience C# and .NET never had much native support for JSON or XML until recently. Newtonsoft's Json.NET became the defacto JSON library because it was [the only](https://www.newtonsoft.com/json/help/html/Introduction.htm) JavaScript library for .NET for some time.

Now, gRPC aside, JSON REST APIs are still fairly ubiquitous amongst back-end APIs. ASP.NET Core 2.1 and earlier actually ships with Json.NET as a dependency. However with .NET Core 3.0 came a new and optimized Json Library offering better performance albeit with a reduced feature set.

#### Newtonsoft vs System.Text
[System.Text.Json was introduced last year as part of .NET Core 3.0](https://devblogs.microsoft.com/dotnet/try-the-new-system-text-json-apis/). As the blog post I have linked suggests, it offers performance improvements over JSON.NET at the cost of a reduced feature set. But without drilling into too much detail yet lets take a look at how much more performant it is with a rudimentary benchmark.


Let's start by trying to benchmark a typical problem in C# - You want to serialize a C# DTO (Data-Transfer-Object) or POCO (Plain-Old-C#-Object) to UTF-8 bytes, either to persist it in a NoSQL db, or to send the data to another Api. Using a C# class to outline your data format is pretty standard, so here's a dummy C# object for us to use:

##### Person.cs
{% highlight csharp %}
public class Person
{
    public Person(string firstName, string lastName, int age, string nemesis, params string[] preferences)
    {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
        Nemesis = nemesis;
        Preferences = preferences;
    }

    public string FirstName {get; set;}
    public string LastName {get;set;}
    public int Age {get;set;}
    public string Nemesis {get; set;}
    public string[] Preferences {get;set;}
}
{% endhighlight %}

As you can see, `Person.cs` is a simple C# class with 5 properties and a constructor. The class would just be used to contain data so it doesn't define any methods.


To serialize this class using Json.NET we can use the example above and define this in a static class & method:
##### NewtonsoftSerialization.cs
{% highlight csharp %}
using System.Text;
using Newtonsoft.Json;

public static class NewtonsoftSerialization
{
    public static byte[] Serialize(Person person)
    {
        var jsonText = JsonConvert.SerializeObject(person);

        return Encoding.UTF8.GetBytes(jsonText);
    }
}
{% endhighlight %}


And the same using System.Text.Json is identical, save for the using statements.
##### SystemTextSerialization.cs
{% highlight csharp %}
using System.Text;
using System.Text.Json;

public static class SystemTextSerialization
{
    public static byte[] SerializeViaJson(Person person)
    {
        var jsonText = JsonSerializer.Serialize(person);

        return Encoding.UTF8.GetBytes(jsonText);
    }
}
{% endhighlight %}


#### Benchmarks

Benchmarking in .NET Core is easily done using BenchmarkDotNet. To get up and running quickly with a benchmark project - create a new console project and import `BenchmarkDotNet` from NuGet.

Now we define a class that calls our serialization methods to benchmark. To do this, we need to add methods to the class that consume the serialization methods and tag them with a `[Benchmark]` attribute. Like so:

##### Utf8Benchmarks.cs
{% highlight csharp %}
using BenchmarkDotNet.Attributes;

[MemoryDiagnoser]
public class Utf8Benchmarks
{
    [Benchmark]
    public byte[] UseNewtonsoft()
    {
        var person = new Person("Fire", "Hawk", 17, "n/a", "fire", "aviation");
        return NewtonsoftSerialization.Serialize(person);
    }

    [Benchmark]
    public byte[] UseSystemText()
    {
        var person = new Person("Omelet", "Ron", 34, "Flute McToot", "woodwind", "4 piece orchestras");
        return SystemTextSerialization.SerializeViaJson(person);
    }
}
{% endhighlight %}

Note that I have added `[MemoryDiagnoser]` to the class. This will instruct BenchmarkDotNet to analyse the memory usage of the benchmarks.


In the `program.cs` file we now add a `using BenchmarkDotNet.Running;` statement and inside the `Main()` method call the static `Run` method, passing a generic argument of the benchmark class.

##### Program.cs
{% highlight csharp %}
using BenchmarkDotNet.Running;

namespace EncodingBenchmarks
{
    class Program
    {
        static void Main(string[] args)
        {
            var runner = BenchmarkRunner.Run< Utf8Benchmarks >();
        }
    }
}
{% endhighlight %}


To run this we now use the terminal and call `dotnet run --configuration "Release"` 
_N.B. dotnet adds certain optimizations to the compiled binary when using the release configuration that would otherwise not be added when using the default DEBUG configuration. Also BenchmarkDotNet requires us to build using the Release Configuration so we don't have much choice anyway!_


### Benchmark Results

```
|         Method |     Mean |     Error |    StdDev |   Median |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|--------------- |---------:|----------:|----------:|---------:|-------:|------:|------:|----------:|
|  UseNewtonsoft | 1.722 us | 0.0358 us | 0.0440 us | 1.709 us | 0.9155 |     - |     - |    1936 B |
|  UseSystemText | 1.276 us | 0.0251 us | 0.0359 us | 1.281 us | 0.3319 |     - |     - |     696 B |
```

For now all we will really consider is the Mean & Allocated output. As you can see from the above whilst Newtonsoft is 500 microseconds slower on average, it allocates nearly 2 kB vs System.Text's 600 Bytes. 

This is a fairly crude benchmark but its surprising how much better System.Text appears to be. 


Both approaches still create a UTF-16 JSON string before they encode and return it as UTF-8. Fortunately System.Text also offers a variation on its serialization method that outputs straight to UTF-8 encoding and saves us this allocation.

##### SystemTextSerialization.cs (cont.)
{% highlight csharp %}
public static class SystemTextSerialization
{
    

    public static byte[] SerializeViaUTF8Json(Person person)
    {
        return JsonSerializer.SerializeToUtf8Bytes(person);
    }
}
{% endhighlight %}

The benchmark results for this method are even better:


|         Method |     Mean |     Error |    StdDev |   Median |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|--------------- |---------:|----------:|----------:|---------:|-------:|------:|------:|----------:|
|  UseNewtonsoft | 1.722 us | 0.0358 us | 0.0440 us | 1.709 us | 0.9155 |     - |     - |    1936 B |
|  UseSystemText | 1.276 us | 0.0251 us | 0.0359 us | 1.281 us | 0.3319 |     - |     - |     696 B |
|  UseSysTxtUtf8 | 1.062 us | 0.0250 us | 0.0404 us | 1.043 us | 0.1984 |     - |     - |     424 B |


The UTF-8 method is almost twice as fast and uses 1/4 of the memory! But hang on, this only amounts to less than a microsecond and 1.5 kB. All in all this isn't really going to be noticed by a user.

But! Remembering the introduction about how allocations impact the garbage collector and this impacts performance - look at the Gen 0, 1 & 2 columns in the table above.
These reflect the Garbage Collections per 1000 operations from their respective generation. The memory allocations from these crude benchmarks don't live beyond Gen 0 but we can see from this table that the we've reduced the number of collections by over four times. Extrapolating this out to a much larger application doing this serialization many times per second and we could see substantial savings.

Some time in the distant future I will investigate:

1. Improving the Benchmark to get less idealized results and analysing the memory profile
2. Drilling into just why System.Text is so much more optimized compared to Json.NET.

- Dan