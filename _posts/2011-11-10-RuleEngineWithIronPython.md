--- 
layout: post
title: Rule Engine using IronPython
comments: true
date: 2011-11-10
---

Currently I'm working on a data processing slash financial application where user can define their own processing rules. Those rules are defined in specially created scripting language - sounds impressive, right? Well it's really interesting piece of code especially for someone like me who never worked before on parsers, tokenizes and all other stuff that is needed to built a compiler/interpreter. But this is also a problem – writing custom language is not a core part of the business. In result language evolved in a direction that was picked as we saw fit at the given moment. That is why we have nice, lean syntactical Frankenstein monster that is working fine with one exception - it's rather slow. 
So conclusion from this situation - if something is not core part of a business and it's pretty important – think twice before you go and build it. Most likely business won’t spare enough time/money/developers to do this right.
 
Realizing this we decided to retire our friendly Frankenstein baby and use Python instead.
 
Python is a dynamic programming language which is getting more traction in recent years. It is used by companies like Google, Yahoo and even NASA. In result of its growing popularity Microsoft decided to create Python implementation for .NET. The project is called [IronPython](http://ironpython.net/) and it was open-sourced not so long ago. Because it is built on .NET using Dynamic Runtime Extensions (DLR) it can be easily integrate with other managed applications.

To embed Python in .NET app it’s enough to reference following assemblies:
 
- Microsoft.Scripting.Metadata.dll

- Microsoft.Scripting.dll

- Microsoft.Dynamic.dll

- IronPython.Modules.dll

- IronPython.dll
 
And type two lines of code, like below, to execute you first script:

<pre><code class="cs">[Test]
public void SimpleExecutionTest()
{
    ScriptEngine engine = Python.CreateEngine();

    dynamic result = engine.Execute(@"2+2");

    Assert.IsTrue(result == 4);  
}
</code></pre>

This is just simple evaluator - execution of something a bit more complicated let say mixing Python method and C# code, can be achieved like this:

<pre><code class="cs">[Test]
public void PassingParameterTest()
{
    ScriptEngine engine = Python.CreateEngine();
    ScriptScope scope = engine.CreateScope();

    string printHello = @"
def PrintHello(name):
	msg = 'Hello ' + name
	return msg";

    ScriptSource source = engine.CreateScriptSourceFromString(printHello, SourceCodeKind.Statements);
    source.Execute(scope);

    var fPrintHello = scope.GetVariable<Func<string, string>>("PrintHello");

    var result = fPrintHello("Michal");          

    Assert.IsTrue(result == "Hello Michal");
}
</code></pre>

After instantiation of necessary objects - ScriptingEngine, ScriptScope etc. about them in a moment, compile a Python method, obtain function delegate from ScriptScope and execute it.
 
There is four main classes that we need to work with IronPython or any language based on DLR for that matter:
 
- ScriptEngine - this is a DLR object that represents language semantics e.g. IronPython, IronRuby etc. It's responsible for executing code.

- ScriptScope - essentially this class represents a namespace - it's used for storing runtime variables. We can execute script in context of multiple ScriptScopes.

- ScriptSource - represents source code and offers a variety of ways to execute or compile the source.

- CompiledCode - represents compiled script - can improve performance in case we want to reuse it.  
 
During integration of Python into the project one of the focuses was ability to reuse library of existing domain functions written in C# without burdening a user with knowledge about references, modules etc. How I choose to solve this problem was using ExpandoObject as a vessel for passing rule delegates into Python script. Let’s look at following example:

<pre><code class="cs">[Test]
public void MixingPythonWithCSharpMethodTest()
{
    ScriptEngine engine = Python.CreateEngine();
    ScriptScope scope = engine.CreateScope();

    // define function
    string add = @"
def Add(a, b):
	return a + b + RULE.GetValue()";

    // prepare our rules
    dynamic dynamicRules = new ExpandoObject();
    var rules = dynamicRules as IDictionary<string, dynamic>;
    rules.Add("GetValue", (Func<int>)GetValue);

    scope.SetVariable("RULE", dynamicRules);

    // compile
    ScriptSource source = engine.CreateScriptSourceFromString(add, SourceCodeKind.Statements);
    CompiledCode compiled = source.Compile();
    compiled.Execute(scope);

    // execute
    dynamic fAdd = scope.GetVariable("Add");

    dynamic result = engine.Operations.Invoke(fAdd, 2, 4);

    Assert.IsTrue(result == 10);
}
</code></pre>

There are two important things here: first script defines function Add which sums two parameters a and b and third value that is obtained from C# function called GetValue(). GetValue in this example is as simple as it can get:

<pre><code class="cs">public int GetValue()
{
    return 4;
}
</code></pre>

Keyword RULE is global variable that we defined as ExpandoObject and passed to ScriptScope which is used to execute script. ExpandoObject holds delegates to functions we want to use in Python method.
It is possible to use dictionary or some other predefined object - instead Expando but first I think that syntax: variable.function(parameters) fit nicely with Python. Secondly we can add properties dynamically which in case of hundreds built in rules can come in handy. Just add some script analyzer and pass only delegates of required functions.
 
All above examples and some more can be found [here](https://github.com/sakowiczm/IronPython-Integration).
