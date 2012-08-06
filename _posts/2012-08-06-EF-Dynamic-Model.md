--- 
layout: post
title: EF Dynamic Model
comments: true
date: 2012-08-06
---

I'm dabbing into EF from time to time mainly to save some time, on non mission critical part of the systems I'm working on. 
And since we have Code First to our disposal - I can do this guilt free for introducing designer generated files to the project. 
What I tried recently is to cheat a bit and force single context to handle different tables that share common structure. I used something like this:

<pre><code class="cs">public class LookupContext : DbContext
{
    private readonly string _tableName;

    public LookupContext(string tableName)
        : base("cn")
    {
        _tableName = tableName;
    }

    public DbSet&lt;LookupItem&gt; Items { get; set; }

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Entity&lt;LookupItem&gt;().Property(c => c.Id).HasColumnName("id");
        modelBuilder.Entity&lt;LookupItem&gt;().Property(c => c.Code).HasColumnName("code");
        modelBuilder.Entity&lt;LookupItem&gt;().Property(c => c.Description).HasColumnName("description");
        modelBuilder.Entity&lt;LookupItem&gt;().HasKey(c => c.Id);

        modelBuilder.Entity&lt;LookupItem&gt;().ToTable(_tableName);
    }
}
</code></pre>

I would pass table name and got DbContext I needed - done and dusted - move on to the next task. Well unfortunately it didn’t work as I envisioned. 
First usage was fine but the moment table name was switched – we were still getting results from the fist table. 
After a bit of digging I found that context type is [cached in the app-domain][1] as an instance of [DbCompiledModel][2]. 
First I was thinking if it would be possible to remove DbContext from the cache without recycling whole app-domain. 
Then I spotted DbContext constructor that accept DbCompiledModel. This lead to following solution:

<pre><code class="cs">var m = new LookupModel();

var model = m.Compile(LookupTable1);
using (var context = new LookupContext(model))
{
    Console.Write(LookupTable1 + " count: ");
    Console.WriteLine(context.Items.Count());
}

model = m.Compile(LookupTable2);
using (var context = new LookupContext(model))
{
    Console.Write(LookupTable2 + " count: ");
    Console.WriteLine(context.Items.Count());
}
</code></pre>

I’m building DbCompiledModel using separate class LookupModel then pass it to my LookupContext where I removed OnModelCreating method. LookupModel looks like this:

<pre><code class="cs">[DbModelBuilderVersion(DbModelBuilderVersion.V4_1)]
public class LookupModel : DbModelBuilder
{
    public LookupModel()
    {
        Configurations.Add(new EntityTypeConfiguration&lt;LookupItem&gt;());

        Entity&lt;LookupItem&gt;().Property(c => c.Id).HasColumnName("id");
        Entity&lt;LookupItem&gt;().Property(c => c.Code).HasColumnName("code");
        Entity&lt;LookupItem&gt;().Property(c => c.Description).HasColumnName("description");
        Entity&lt;LookupItem&gt;().HasKey(c => c.Id);                
    }

    public DbCompiledModel Compile(string tableName)
    {
        Entity&lt;LookupItem&gt;().ToTable(tableName);

        return Build(new DbProviderInfo("System.Data.SqlClient", "2008")).Compile();
    }
}
</code></pre>

One thing we should remember using this solution – it might not be the best option for models with large number of entities. 
Context creation is expensive operation. You can of course make your own caching mechanism and store compiled models in a dictionary where table name would be a key. 
In my case with extremely simple model it seems a bit excessive. Full code example can be found [here][3].

[1]: http://blog.oneunicorn.com/2011/04/15/code-first-inside-dbcontext-initialization
[2]: http://msdn.microsoft.com/en-us/library/system.data.entity.infrastructure.dbcompiledmodel%28v=vs.103%29.aspx
[3]: https://github.com/sakowiczm/EF-Dynamic-Model