--- 
layout: post
title: ConditionalWeakTable Class
comments: true
date: 2018-03-29
---

[ConditionalWeakTable][1] is not new but unknown to me piece of functionality offered by C#. It was introduced in C# 4 and allows thread-safe, association of additional data with existing class. There is no need to modify class implementation, it can be sealed or supplied by 3rd party library. Let's consider following example:

``` csharp
using System.Runtime.CompilerServices;

public sealed class Data
{
    public int Value1 { get; set; }
}

public class ExtraValues
{
    public int Value2 { get; set; }
}

public static class ExtraValuesProvider
{
    private static ConditionalWeakTable<Data, ExtraValues> _table = new ConditionalWeakTable<Data, ExtraValues>();

    public static void SetValue2(this Data data, int value)
    {
        _table.GetOrCreateValue(data).Value2 = value;
    }

    public static int GetValue2(this Data data)
    {
        return _table.GetOrCreateValue(data).Value2;
    }

    public static void PrintValues(this Data data)
    {
        Console.WriteLine($"Value1: {data.Value1}, Value2: {data.GetValue2()}");
    }
} 
``` 

Objects above:

* `Data` - class we want to extend.
* `ExtraValues` - additional information we want to store.
* `ExtraValueProvider` - object that facilitates linking Data with ExtraValues and provides extension methods for data manipulation.

We can use it like this:

``` csharp
static void Main(string[] args)
{
    var data = new Data();
    data.Value1 = 1;
    data.SetValue2(2);

    int i = data.GetValue2();

    data.PrintValues();
}
``` 

`ConditionalWeakTable` is using [WeakReference][2] to link classes so when the object instance is garbage collected, any attached values are automatically cleaned up as well. 

Extra points to note:

* In the code snippet we have method `PrintValues` - extension method `ToString()` would fit here nicer - unfortunately it's not possible - [as instance methods have precedence over extension methods][3]. 
* It would also be cleaner to replace `GetValue2` and `SetValue2` methods with property - as above currently it's not possible to create [extension properties][4]. 
* [Mixin][5] - new term for me - describing class that contains methods for use by other classes without using inheritance. In above case I believe ExtraValues and ExtraValuesProvider can be called a mixin. 


[1]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2?view=netcore-2.0
[2]: https://docs.microsoft.com/en-us/dotnet/api/system.weakreference?view=netcore-2.0
[3]: https://stackoverflow.com/questions/4982479/how-to-create-an-extension-method-for-tostring
[4]: https://stackoverflow.com/questions/619033/does-c-sharp-have-extension-properties
[5]: https://en.wikipedia.org/wiki/Mixin