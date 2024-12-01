---
title: Making an ASP.NET Core Endpoint more event-driven
date: 2022-02-11 00:00:00 +/-TTTT
categories: [dotnet, csharp]
tags: [architecture]     # TAG names should always be lowercase
---

## The problem

You may have found yourself in this situation: you have an API endpoint that involves a long running process, the result of which you don't actually need to know. You can't just return your response early, because that long running process will end prematurely, but you also don't want your request staying open for potentially minutes at a time, or even worse timing out.

This problem usually doesn't appear overnight, and is often the result of years of evolving processes without evolving your architecture. While event driven architectures are nothing new, the adoption of more event driven principles has increased in recent years with the explosion of cloud computing making this easier than ever. Services such as Azure Functions, coupled with a Service Bus or Event Grid, mean that building fully event driven applications, or even just applications with a few event driven components, is ridiculously easy on paper.

But imagine that you're in a situation that I recently found myself in: a mature API with a tangled mess of code with virtually no tests. Without tests I couldn't verify exactly what the code was doing, or even if it was doing it correctly. All I knew was that I had an API endpoint that was now timing out and crashing a crucial business process. Refactoring this into a new Azure Function and getting it through regression testing would take too long, and the business didn't have time to spare.

## Enter IHostedService

Hosted services have been around in ASP.NET Core since version 2.0, and run in the background to your web host. They were created as a way to run background tasks in your web application, and can even be set to run on a schedule. These will run for as long as the application is running, and are completely decoupled from incoming requests. Obviously I can't reveal the actual code I applied this to, so for the sake of explaining the pattern I'll stick with a nice simple example that pretty much anybody will understand.

For most background tasks, the Microsoft provided BackgroundService implementation will be sufficient to inherit from, so we will use the following as a starting point:

```cs
public class OrderReceivedEmailDispatcher : BackgroundService
{
    private readonly ILogger<OrderReceivedEmailDispatcher> _logger;

    public OrderReceivedEmailDispatcher(
        ILogger<OrderReceivedEmailDispatcher> logger)
    {
        _logger = logger;
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("OrderReceivedEmailDispatcher is starting");

        stoppingToken.Register(() =>
        {
            _logger.LogInformation("OrderReceivedEmailDispatcher is stopping");
        });
        
        while (!stoppingToken.IsCancellationRequested)
        {
            // Functionality coming soon!
        }
        
        _logger.LogInformation("OrderReceivedEmailDispatcher is stopping");

        return Task.FromCanceled(stoppingToken);
    }
}
```
```cs
// Program.cs

// To register the hosted service in the DI container:
builder.Services.AddHostedService<OrderReceivedEmailDispatcher>();
```

## Creating a Publisher

In order for our background service to be of any use, we need to feed it some information to process. Remember, we can't access the request pipeline because this is a background service, so a pattern I really like is to create a class responsible for publishing this information:

```cs
public class NewOrderPublisher : INewOrderPublisher
{
    /// <summary>
    /// Raises an event to signify that a new order is available.
    /// </summary>
    public event EventHandler<Order>? NewOrder;
    
    /// <summary>
    /// Takes an order and publishes it via the event handler.
    /// </summary>
    /// <param name="order">The new order to publish.</param>
    public void Publish(Order order)
    {
        NewOrder?.Invoke(this, order);
    }
}
```

You may be wondering at this point why I decided to use an event, and the answer is quite simple: by doing so, I am able to decouple the process of raising the event from the process of responding to the event. We can now connect our publisher to our background service by injecting it into the service via the constructor and creating a delegate to respond to the event:

```cs
public class OrderReceivedEmailDispatcher : BackgroundService
{
    private readonly ILogger<OrderReceivedEmailDispatcher> _logger;
    private readonly INewOrderPublisher _publisher;
    private readonly ICustomerService _customerService;
    private readonly IEmailService _emailService;

    public OrderReceivedEmailDispatcher(
        ILogger<OrderReceivedEmailDispatcher> logger, 
        INewOrderPublisher publisher, 
        ICustomerService customerService, 
        IEmailService emailService)
    {
        _logger = logger;
        _publisher = publisher;
        _customerService = customerService;
        _emailService = emailService;
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("OrderReceivedEmailDispatcher is starting");

        stoppingToken.Register(() =>
        {
            _logger.LogInformation("OrderReceivedEmailDispatcher is stopping");
        });
        
        while (!stoppingToken.IsCancellationRequested)
        {
            _publisher.NewOrder += async (_, order) =>
            {
                var customer = await _customerService.GetCustomer(
                    order.CustomerId);

                if (customer != null)
                {
                    await _emailService.Send(
                        customer.Email, 
                        $"Order {order?.OrderReference} Received.");
                }
            };
        }
        
        _logger.LogInformation("OrderReceivedEmailDispatcher is stopping");

        return Task.FromCanceled(stoppingToken);
    }
}
```

A very important thing to note here is that we need to register our publisher class as a Singleton, because we need to ensure that the same instance is being used throughout our application. Otherwise, we'll have a bunch of events being raised with nothing listening in.

## Putting it all together

The final piece of our puzzle is to connect our publisher to our controller, which as you can imagine is as simple as injecting it via the constructor:

```cs
public class OrderController : ControllerBase
{
    private readonly INewOrderPublisher _publisher;

    public OrderController(INewOrderPublisher publisher)
    {
        _publisher = publisher;
    }

    [HttpPost]
    public IActionResult OrderCreated(Order order)
    {
        _publisher.Publish(order);
        return Ok();
    }
}
```

Obviously in the real world this controller would be a lot more substantial but for the sake of demonstrating this pattern I kept it as simple as possible.

## Final Thoughts

By adopting this approach, we are able to create an event driven process in our application that will return a response to the client to signify that their request has been acknowledged, instead of waiting around for the finished product. This is a bit of a mindset shift from the traditional request/response model, but it can be incredibly powerful if you don't actually need anything from the finished product.

A major consequence of this approach, however, is that you will no longer know if the process has failed via your HTTP response. This is an approach that will require you to really think about monitoring and observability in your applications. Something I like to do is raise a custom event in Application Insights and set up an alert in Azure to email me if this event is ever raised. That being said, I think for the simplicity and extensibility that this approach brings you, having to maybe make a conscious decision about monitoring isn't too big a price to pay.

Event driven architectures have a great role to play in an increasingly microservice-heavy landscape, and what I have covered here is barely the tip of the iceberg, but I hope it is helpful for at least one other person.
