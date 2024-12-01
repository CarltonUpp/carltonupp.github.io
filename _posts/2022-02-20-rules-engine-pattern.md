---
title: The Rules Engine Pattern
date: 2022-02-20 00:00:00 +/-TTTT
categories: [dotnet, csharp]
tags: [design-patterns]     # TAG names should always be lowercase
---


## Introduction

As professional or aspiring software engineers, we are usually tasked with turning business rules into something that a computer understands. We model our problem domain using classes, and write business logic to reflect real world rules that exist outside of the codebase. When these business rules change in the real world, they must also change in the code that represents them, and this is where the real complexity of our field lies.

Writing new code is the easy part, it's changing your existing code that holds most of the challenges.

## Defining Good Code

A common sentiment among software engineers is that good code is code that can be changed easily. Another one is that one should endeavour to write code as if the next person to work on it will be a murderous psychopath. Both are different takes on the same principle.

If you only write code that never ever changes, you might as well stop reading here because this post isn't for you. For those of us that work in a business with ever evolving rules, I'd like you to consider the following bit of code:

```csharp
public class ShippingCalculator
{
    public decimal Calculate(BasketDetails details)
    {
        if (details.Customer.IsOverseas)
        {
            if (details.BasketTotal >= 200)
            {
                return 0m;
            }

            return 49.99m;
        }
        else
        {
            if (details.BasketTotal >= 100)
            {
                return 0m;
            }

            return 19.99m;
        }
    }
}
```
It's a fairly simple bit of code used for calculating the price of shipping for an ecommerce website. The rules are:
* Domestic shipping is £19.99
* International shipping is £49.99
* Shipping is free if the basket total meets the threshold of £100.00 for domestic orders or £200.00 for international orders

Lets say the business introduced a new rule to give half price domestic shipping during the month of December: think about how you would change this code to accommodate this. If we're going with the simple solution, we could add something like the following to our domestic shipping branch of the if statement:

```csharp
if (DateTime.Now.Month == 12)
{
    return 9.99m;
}
```

Now imagine they extended this offer to international orders as well. At this point the code is becoming complicated, and is in desperate need of refactoring. Sure, you could use some of the nicer features of C## to clean it up to some extent, but there's only so far that could take you.

## The Open-Closed Principle

I've you've had a job interview in the last decade or so, you've probably had to explain one or more of the SOLID principles. One of the less understood, but incredibly important principles is the **open-closed principle**, which states that adherent code should be open to extension, but closed to modification. Like most of the SOLID principles, this is left intentionally vague to give conference speakers something to waffle on about, but the core of the principle is that **you should be able to add functionality as an extension to the existing code, not a replacement of it**.

After hearing this explanation, do you think that the above code is adherent to this principle? In order to add a new rule to the shipping calculation algorithm, we had to add more code to the already messy code, so the answer is no. Every change to this code runs the risk of introducing a bug, and let's be honest: code that looks like this is almost never covered by unit tests, or at least not by up-to-date ones, so we can't confidently make changes to it.

## Applying the Rules Engine pattern

Instead of giving a theoretical overview of the Rules Engine pattern, I'll attempt to explain it by just demonstrating the code we will be refactoring our existing implementation with. 

The first thing we need is an interface to define what a **Rule** looks like in the context of our system:
```csharp
public record ShippingCalculatorRuleResult(bool Applied, decimal Shipping);
public record ShippingCalculatorRuleFailedResult() : ShippingCalculatorRuleResult(false, 0m);
public record ShippingCalculatorRuleSuccessResult(decimal Shipping) : ShippingCalculatorRuleResult(true, Shipping);

public interface IShippingCalculatorRule
{
    ShippingCalculatorRuleResult Calculate(BasketDetails basket);
}
```

And then we need a class that represents our Rules Engine, which take a collection of rules via the constructor and has a method for delivering the computed result of all rules being executed:

```csharp
public class ShippingCalculatorRulesEngine
{
    private readonly IReadOnlyCollection<IShippingCalculatorRule> _rules;

    public ShippingCalculatorRulesEngine(IReadOnlyCollection<IShippingCalculatorRule> rules)
    {
        _rules = rules;
    }

    public decimal CalculateShipping(BasketDetails basket)
    {
        /* We want to return the lowest shipping price
            that the customer is entitled to.*/
        return _rules
            .Select(r => r.Calculate(basket))
            .Where(r => r.Applied)
            .Min(r => r.Shipping);
    }
}
```
Now let's create some rules:
```csharp
public class InternationalShippingRule : IShippingCalculatorRule
{
    public ShippingCalculatorRuleResult Calculate(BasketDetails basket)
    {
        return basket.Customer switch
        {
            { IsOverseas: true } => new ShippingCalculatorRuleSuccessResult(49.99m),
            _ => new ShippingCalculatorRuleFailedResult()
        };
    }
}

public class BasketTotalRule : IShippingCalculatorRule
{
    public ShippingCalculatorRuleResult Calculate(BasketDetails basket)
    {
        return basket switch
        {
            { Customer.IsOverseas: true, BasketTotal: >= 200.00m } => new ShippingCalculatorRuleSuccessResult(0m),
            { BasketTotal: >= 100.00m } => new ShippingCalculatorRuleSuccessResult(0m),
            _ => new ShippingCalculatorRuleFailedResult()
        };
    }
}

public class HalfPriceDecemberShippingRule : IShippingCalculatorRule
{
    public ShippingCalculatorRuleResult Calculate(BasketDetails basket)
    {
        return DateTime.Now.Month switch
        {
            12 when basket.Customer.IsOverseas => new ShippingCalculatorRuleSuccessResult(24.99m),
            12 => new ShippingCalculatorRuleSuccessResult(9.99m),
            _ => new ShippingCalculatorRuleFailedResult()
        };
    }
}
```

And finally we can consume our rules engine result in our Shipping Calculator class:

```csharp
public class ShippingCalculator
{
    private readonly ShippingCalculatorRulesEngine _shippingCalculatorRulesEngine;

    public ShippingCalculator()
    {
        /* We can use reflection to find all rules in the current project.
            I may cover other ways of doing this in a future post.*/
         
        var ruleType = typeof(IShippingCalculatorRule);
        IReadOnlyCollection<IShippingCalculatorRule> rules = GetType().Assembly.GetTypes()
            .Where(p => ruleType.IsAssignableFrom(p) && !p.IsInterface)
            .Select(r => Activator.CreateInstance(r) as IShippingCalculatorRule)
            .ToList()
            .AsReadOnly()!;

        _shippingCalculatorRulesEngine = new ShippingCalculatorRulesEngine(rules);
    }

    public decimal Calculate(BasketDetails basket)
    {
        return _shippingCalculatorRulesEngine.CalculateShipping(basket);
    }
}
```
So based on this code, we have the following components in a rule engine implementation:

* A contract that all rules must implement
* Implementations of rules
* An engine that executes all rules and returns the compiled value or result

All of the other stuff like the result types are just an implementation detail. Realistically all you need are the three components listed above (along with a way of getting the rules into the rules engine) and you are ready to go. 

## Adding a new Rule

Now that we have a working Rules Engine implementation, we are able to add new rules faster than ever before. Let's say the business decides to halve the free shipping threshold on a customer's birthday, we can just add a new rule implementation and the rest is taken care of:

```csharp
public class BirthdayCouponShippingRule : IShippingCalculatorRule
{
    public ShippingCalculatorRuleResult Calculate(BasketDetails basket)
    {
        if (basket.Customer.DateOfBirth != DateTime.Now.Date) 
            return new ShippingCalculatorRuleFailedResult();
        
        return basket switch
        {
            { Customer.IsOverseas: true, BasketTotal: >= 100.00m } => new ShippingCalculatorRuleSuccessResult(0m),
            { BasketTotal: >= 50.00m } => new ShippingCalculatorRuleSuccessResult(0m),
            _ => new ShippingCalculatorRuleFailedResult()
        };
    }
}
```

Adding a rule is that simple, and due to these rules being pure functions it makes them an absolute dream to test! In case you aren't up on your functional programming theory, a pure function is one that has no side effects - meaning you can pass the same values into the function a thousand times and get the exact same result.

## Conclusion

By utilising the Rules Engine pattern, we are able to model complex business processes as a series of rules and significantly reduce our cyclomatic complexity. While I tend to err on the side of caution when it comes to using design patterns, I'd recommend this pattern if your code exhibits the following traits:
* Lots of nested if statements
* Frequently being updated
* Is responsible for providing a returned value

Making your code easier to work with means you can change your code with less friction and less friction means you are able to deliver faster, which can sometimes be the difference between a business succeeding or failing.
