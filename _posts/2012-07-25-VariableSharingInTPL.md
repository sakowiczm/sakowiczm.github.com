--- 
layout: post
title: "Variable 'sharing' in TPL"
comments: true
date: 2012-07-25
---

I was debugging multithreaded piece of code and I got to the bit that was expressing something like this:

<pre><code class="cs">private static void RaceConditionIssue()
{
    int result = 0;
    var tl = new List&lt;Task&gt;();

    for (int i = 0; i &lt; 10; i++)
    {
        Task t = Task.Factory.StartNew(() =&gt;
        {
            result++;
        });

        tl.Add(t);
    }

    Task.WaitAll(tl.ToArray());
    Console.WriteLine(result);
}</code></pre>

This is example of how easy it is to introduce race condition issues to your application. To better understand the problem let’s look at code that was generated for delegate under Reflector:

<pre><code class="cs">[CompilerGenerated]
private sealed class &lt;&gt;c_DisplayClass2
{
    // Fields
    public int result;

    // Methods
    public void &lt;Main&gt;b_0()
    {
        this.result++;
    }
}
</code></pre>
.NET is not using variable from the static method its creating its local representation. Having said that imagine that first two task start at the same time with initial result value equal to 0. Both of them increase value to 1 (local variables). Then third thread starts and increment result value from 1 to 2. We executed three threads but result is 2 - incorrect. Output of above code is random depending on order/speed of separate tasks.

To avoid this kind of issues is advised to use [Task Parameters][1]. The above example is not the best one for them but I think you got the point. One more thing - why the whole code is not blowing up when multiple threads are reading/writing result variable? - .NET primitives like int are thread-safe and reads/writes are atomic.

[1]: http://msdn.microsoft.com/en-us/library/dd537609 "Task Parameters"
