--- 
layout: post
title: Agatha and Request/Response pattern
comments: true
date: 2011-10-29
---

Request/Response pattern is very simple pattern that can make our life easier when working with wcf services. The idea is to wrap all service operation parameters in one object that we call a request and return an object that we call a response. There is few advantages of such approach, the main ones (at least for me) are:

- You can inherit request and responses from base objects where you can put some common properties like user information, security token or error message.

- Simplifies versioning of a service

- Generated proxy is easier to read and understand

Simple right? Sure – that is why I was surprised when I’ve read about framework that was described as a ‘Request/Response Service Layer’. Why the strait forward pattern needs a framework? Is this one of those projects that were created just for sake of creating it? I was almost ready to close the page when I saw a little bit of code that I sub conscience liked:

<pre><code class="cs">namespace Sample.ServiceLayer.Handlers
{
    public class HelloWorldHandler : RequestHandler<HelloWorldRequest, HelloWorldResponse>
    {
        public override Response Handle(HelloWorldRequest request)
        {
            var response = CreateTypedResponse();
            response.Message = "Hello World!";
            return response;
        }
    }
}</code></pre>

One class one operation? Interesting I must admit most services which I was working with – looked like bunch different procedures putted in the same file – procedural, ugly hard to read and understand. Is this possibly a solution?

Please meet Agatha – she will make your life easier or not – depends your preferences. So what this framework is doing? – simply it’s request dispatcher. There is predefined contract with one operation:

<pre><code class="cs">public interface IRequestProcessor : IDisposable
{
    Response[] Process(params Request[] requests);
}</code></pre>

All messages are send to this operation – and Agatha based on request object type is routing it to specific handler. Basic sample can be found here. So what we gain:

proper code separation for service operations – now we have handlers
we can publish only one service for all implemented handlers
no need to update client side proxy when service is changed
authentication, authorization, error logging – can be done just in one place
we can batch requests for traffic optimization – so ‘chatty’ services are not a problem anymore
framework handles client and server endpoints with minimal wcf configuration
So it is nice simple and easy – but only when both client and server are using Agatha. We can still consume service in other languages but it will be awkward – request types will indicate what operation we want to call. One more issue that was actually mentioned by one of my colleagues is lack of clearly defined interface which will make service even more difficult to use by 3rd party.

In summary interesting approach for handling services. Especially if you want to use them for internal clients that you can control. But I would think twice if I had to develop something for greater audience.

You can find Agatha on [GitHub](https://github.com/davybrion/Agatha). Example application is described [here](http://davybrion.com/blog/2009/11/hello-world-with-agatha/), and [here](http://davybrion.com/blog/2009/11/requestresponse-service-layer-series/) you can find series of post describing framework in more details.