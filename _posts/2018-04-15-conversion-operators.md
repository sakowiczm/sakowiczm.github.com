--- 
layout: post
title: Conversion operators
comments: true
date: 2018-04-15
---

[Conversion operators][1] allow us to write following code:

``` csharp
Money m = 121.55m;
```

This is basically custom conversion from `decimal` to `Money` type. It's C# basics available since its inception - but honestly, I had to look it up. I believe it's very rarely used and I don't remember when I last saw a piece of code using this technique. Custom conversion are implemented as class [operator][2] and they need to be declared as `static`. Keywords [explicit][3] and [implicit][4] are controlling if we have to use casting or not. 

For example:

``` csharp
public class Money
{
    public decimal Amount;

    public Money(decimal amount)
    {
        Amount = amount;
    }

    // explicit decimal to Money conversion operator
    public static explicit operator Money(decimal d)
    {
        return new Money(d);
    }

    // implicit Money to decimal conversion operator
    public static implicit operator decimal(Money m)
    {
        return m.Amount;
    }
}

class Program
{
    static void Main(string[] args)
    {
        decimal a = 121.55m;

        // explicit conversion
        Money b = (Money)a;

        // implicit conversion - no cast needed
        decimal c = b;
    }
}
```

[1]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/using-conversion-operators
[2]: https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/operator
[3]: https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/explicit
[4]: https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/implicit


    