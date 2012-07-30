--- 
layout: post
title: SQL Query differences
comments: true
date: 2012-07-30
---
 
Recently I had an issue where exactly the same query was working fine form SQL Management Studio but was extremely slow in my application. I did some digging and as expected everything comes to different execution plans. In SQL Server each query can have multiple query plans – separate plan for each query execution with different connection settings. 
 
By default connection settings are set like this:

<center>
<img title="Default connection settings" src="/img/posts/2012-07-30-SQLQueryDifferences1.png" alt="Default connection settings" />
</center>
 
The ARITHABORT flag is always different – so if you copy a query form application and execute it inside Management Studio – most likely different plan will be generated - because of different input parameters (parameter sniffing).
 
To be able to reuse existing plan (or increase chances to do so) you can use:
 
[SET ARITHABORT OFF][1]
 
or go and switch of below flag in SQL Management Studio:

<center>
![SET ARITHABORT setting in MS SQL Management Studio](/img/posts/2012-07-30-SQLQueryDifferences2.png)
</center>

[1]: http://msdn.microsoft.com/en-us/library/aa259212(v=sql.80).aspx 