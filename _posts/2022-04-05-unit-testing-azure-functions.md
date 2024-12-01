---
title: Unit Testing HTTP Trigger Azure Functions running as isolated processes
date: 2022-04-05 00:00:00 +/-TTTT
categories: [azure, azure-functions]
tags: [cloud-computing, serverless]     ## TAG names should always be lowercase
---

## Introduction

Last December, Microsoft released the long-awaited .NET 6 Release, which came with a wealth of features- my favourite of which being that it's an LTS version. This meant that we can finally migrate our codebases at work from .NET Core 3.1 and gain access to everything that came with not just .NET 6, but .NET 5 as well.

Something that came in the prior release that I hadn't had the chance to work with yet was the new, isolated-process hosting model for Azure Functions. This blog post isn't going to explain every last detail about this new hosting model, but the general idea is that you have access to the standard dependency injection and middleware capabilities of .NET, greater control over the process and the function app runs as a Console application instead of as a Class Library.

At work, we had just finished upgrading our APIs to .NET 6 in the prior sprint and it was now time for me to look at upgrading our function apps to .NET 6 and explore adopting the new hosting model. Migrating the function itself was no big deal but we also had to fix the tests that were now broken due to API changes. After looking online for the best part of a day and finding no resources on how to test these new functions, I ended up doing some digging into the underlying packages and figuring out for myself how to fix these broken tests.

## Our Example Function

In the interest of my employer not taking me to court, I will use the following example for demonstration purposes. It's a pretty simple function: it takes a POST request with a body containing a name property, then returns "Hello, name" as a response with a status of `200 OK`. If the request body is invalid, it will return a `400 Bad Request`.

```csharp
public class GreeterHttpFunction
{
    [Function("http-greeter")]
    public async Task<HttpResponseData> Greet(
        [HttpTrigger(AuthorizationLevel.Function, "post")] 
        HttpRequestData req)
    {
        var response = req.CreateResponse(req.Body.TryParseJson<HttpGreeterRequest>(out var body) switch
        {
            false => HttpStatusCode.BadRequest,
            true when string.IsNullOrWhiteSpace(body?.Name) => HttpStatusCode.BadRequest,
            true => HttpStatusCode.OK
        });

        if (response.StatusCode is HttpStatusCode.OK) 
            await response.WriteStringAsync($"Hello, {body?.Name}!");

        return response;
    }
}

public static class HttpRequestDataExtensions
{
    public static bool TryParseJson<TOutputType>(this Stream @this, out TOutputType? result)
    {
        using var streamReader = new StreamReader(@this, encoding: Encoding.UTF8);
        var json = streamReader.ReadToEnd();

        if (string.IsNullOrWhiteSpace(json))
        {
            result = default;
            return false;
        }

        try
        {
            result = JsonConvert.DeserializeObject<TOutputType>(json);
            return true;
        }
        catch (Exception ex) when(ex is JsonSerializationException or JsonReaderException)
        {
            result = default;
            return false;
        }
    }
}
```

## Test Project Setup

For our test project, I'm using xUnit, Shouldly and Moq but you are free to use whichever you prefer. The only hard requirements for packages in our test project are:

* Microsoft.Azure.Functions.Worker
    
* Microsoft.Azure.Functions.Worker.Extensions.Http
    

To get us quickly set up with our test class, you can use the following snippet:

```csharp
public class HttpGreeterTests : IClassFixture<GreeterHttpFunction>
{
    private readonly GreeterHttpFunction _sut;

    public HttpGreeterTests(GreeterHttpFunction sut)
    {
        _sut = sut;
    }
}
```

## Mocking HttpRequestData and HttpResponseData

Testing these new functions gets slightly annoying in the sense that the `HttpRequestData` and `HttpResponseData` classes exposed by the isolated-process sdk are both abstract, and the default implementations are internal classes, meaning they are unusable outside of the package housing them.

I did a great deal of experimentation with creating a mock implementation of `HttpRequestData` and I ended up with the following:

```csharp
public sealed class MockHttpRequestData : HttpRequestData
{
    // No behaviour is actually needed from this.
    private static readonly FunctionContext Context = Mock.Of<FunctionContext>();
    
    public MockHttpRequestData(string body) : base(Context)
    {
        // I added the body parameter just to clean up boilerplate.
        var bytes = Encoding.UTF8.GetBytes(body);
        Body = new MemoryStream(bytes);
    }

    public override HttpResponseData CreateResponse()
    {
        // The actual response creation is done via extension methods
        return new MockHttpResponseData(Context);
    }

    public override Stream Body { get; }
    public override HttpHeadersCollection Headers { get; }
    public override IReadOnlyCollection<IHttpCookie> Cookies { get; }
    public override Uri Url { get; }
    public override IEnumerable<ClaimsIdentity> Identities { get; }
    public override string Method { get; }
}
```

Our response class is really simple - it just implements the required members and that is it:

```csharp
public sealed class MockHttpResponseData : HttpResponseData
{
    public MockHttpResponseData(FunctionContext context) : base(context)
    {
    }

    public override HttpStatusCode StatusCode { get; set; }
    public override HttpHeadersCollection Headers { get; set; }
    public override Stream Body { get; set; } = new MemoryStream();
    public override HttpCookies Cookies { get; }
}
```

## Writing some Tests

Armed with our mocked up request and response classes, we can now begin writing some tests for our function. To make these tests a bit more concise, I created a utility extension method for reading the response body as a string:

```csharp
public static async Task<string> GetResponseBody(this HttpResponseData response)
{
    response.Body.Seek(0, SeekOrigin.Begin);
    using var reader = new StreamReader(response.Body);
    return await reader.ReadToEndAsync();
}
```

We'll start by covering all paths that will produce a bad request response code:

```csharp
    [Fact]
    public async Task InvalidJsonBodyReturnsBadRequest()
    {
        var request = new MockHttpRequestData("{invalid}");
        var response = await _sut.Greet(request);
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
    
    [Fact]
    public async Task EmptyNameInBodyReturnsBadRequest()
    {
        var request = new MockHttpRequestData("{'name': ''}");
        var response = await _sut.Greet(request);
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
    
    [Fact]
    public async Task EmptyBodyReturnsBadRequest()
    {
        var request = new MockHttpRequestData("");
        var response = await _sut.Greet(request);
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
```

Notice how these tests are identical aside from the json body? We can refactor this to use a Theory and reduce repeated code, and while we're at it I'm going to change our assertions to use Shouldly.

```csharp
    [Theory]
    [InlineData("{invalid}")]
    [InlineData("{'name': ''}")]
    [InlineData("")]
    public async Task InvalidRequestBodyReturnsBadRequest(string body)
    {
        var request = new MockHttpRequestData(body);
        var response = await _sut.Greet(request);
        response.StatusCode.ShouldBe(HttpStatusCode.BadRequest);
    }
```

Lets finish off our tests by making sure a valid JSON body will return `200 OK`:

```cs
    [Theory]
    [InlineData("John Wick")]
    [InlineData("Bryan Mills")]
    [InlineData("Katniss Everdeen")]
    public async Task ValidRequestBodyReturnsOkWithMessage(string name)
    {
        var request = new MockHttpRequestData($"{ 'name': '{name}' }");
        var response = await _sut.Greet(request);
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        response.StatusCode.ShouldBe(HttpStatusCode.OK);

        var message = await response.GetResponseBody();
        message.ShouldBe($"Hello, {name}!");
    }
```

And now if you pray hard enough these tests should all be passing! If you have any issues with the code in this blog post, please see the repo hosting the code [here](https://github.com/carltonupp/TestingAzureFunctions)

## Conclusion

Despite the initial problem I had finding information on the topic, I think I managed to find a fairly simple and effective way to test Azure Functions in this new hosting model. I hope this information finds someone in a similar situation.

While I generally prefer integration tests for testing http resources, unit tests are still better than no tests. I intend to do a post on integration testing as soon as I figure out how to do it myself.

Until next time!
