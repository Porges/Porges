---
date: 2000-01-02
tags:
  - software-design
title: Ignore Null Strategies
---
_There’s no such thing as a null strategy!_

Here is a simple refactoring/micro-pattern that can help remove a little complexity from classes that use passed-in behaviours/strategies/handlers/whatever you want to call them.[^1] The examples given here may seem rather abstract or “enterprisey”, but it’s something that comes up repeatedly when developing software.

[^1]: Here I’m going to use strategy in a broad sense, to basically include any behaviour that is injected into a class. Other people might call these collaborators (although I’d apply that to the implementations of the strategy and not the strategy abstraction itself), or something different.
### The problem: Checking strategies for nullness

Below is a very simplified example of a consumer (the class `ProducerExample`) of the `IProducer` strategy. The consumer explicitly checks if the strategy is null before invoking it, and if it _is_ null, it produces some default value instead.

This extra null check gets in the way of what the class is actually doing and introduces an extra code path that is not wanted (or needed).

```csharp
interface IProducer
{
    Result Produce();
}

class ProducerExample
{
    readonly IProducer _producer;

    // constructor elided...

    public Result Act()
    {
        if (_producer == null)
        {
            return Result.Default;
        }

        return _producer.Produce();
    }
}
```

(The producer here is deliberately simple—in a real codebase this could take any number of arguments.)

One situation where this kind of code might appear is when a previously-untested class is being tested for the first time. In this case, dummy values might be inserted in order that `null` can be passed as the strategy at test time and you’ll still get results instead of a `NullReferenceException`. This kind of setup is dangerous, as inadvertently passing `null` in the production code will end up generating these dummy values! 💥

Here’s a very similar example, using a strategy that doesn’t have a result (which I’ll call a “handler”). This is checked for nullness before it is invoked for its side-effects:

```csharp
interface IHandler
{
    void Handle();
}

class HandlerExample
{
    readonly IHandler _handler;

    // constructor elided...

    public void Run()
    {
        if (_handler == null)
        {
            // some other implementation
        }
        else
        {
            _handler.Handle();
        }
    }
}
```

### Solution: Don’t check strategies for nullness!

Instead of explicitly checking for nullness, assume that the strategy is not null (you can enforce this in the constructor) and migrate any “default action” into a new implementation of the strategy.

Then, anywhere you were previously passing in `null`, you can instead pass an instance of your shiny new implementation.

For the producer example, we can move the default implementation into a new class, and remove the null check:

```csharp
class DefaultProducer : IProducer
{
    public Result Produce() => Result.Default;
}

class ProducerExampleRevised
{
    [NotNull] readonly IProducer _strategy;

    // constructor elided...

    public Result Run()
    {
        return _strategy.Produce();
    }
}
```

(The `[NotNull]` annotation here is something supported by Resharper via the [Resharper.Annotations](https://www.nuget.org/packages/JetBrains.Annotations) package.)

If we need to change the result of the producer depending on what the consumer is, we can use a variation on this, and create an implementation that stores any value we want:

```csharp
class ConstantProducer : IProducer
{
    public Result Value { get; }
    
    public ConstantProducer(Result value)
    {
        Value = value;
    }

    public Result Produce() => Value;
}
```

In the “handler” case, we can create a [‘null object’ implementation](https://en.wikipedia.org/wiki/Null_Object_pattern) that does nothing, since we don’t need to produce a result:

```csharp
class NullHandler : IHandler
{
    public void Handle() { }
}

class HandlerExampleRevised
{
    [NotNull] readonly IHandler _handler;

    // constructor elided...

    public void Run()
    {
        _handler.Handle();
    }
}
```

With the refactored code, the consuming classes are cleaner (less code, lower cyclomatic complexity), and we have extracted a “default” implementation, which could potentially be used by other consumers.

At the same time, we have created some useful additional implementations that we can use in unit tests! The “Default” or “Constant” producers are useful when providing canned data to classes that are being tested, and “Null” handlers are useful when ignoring part of the behaviour of a class in order to test other parts.

#### Variation: Function-oriented implementation

For single-method interfaces such as those above we can replace them with delegates. This can lead to much cleaner code.

For producers:

```csharp
delegate Result Producer();

static class Producers
{
    // Default is a Producer
    public static Result Default() => Result.Default;

    // Constant returns a Producer, given an argument
    public static Producer Constant(Result r) => () => r;
}

class FunctionalProducerExample
{
    [NotNull] readonly Producer _strategy;

    // constructor elided...

    public Result Run()
    {
        return _strategy();
    }
}
```

And for handlers:

```csharp
delegate void Handler();

static class Handlers
{
    // Null is a Handler
    public static void Null()
    {
    }
}

class FunctionalHandlerExample
{
    [NotNull] readonly Handler _handler;

    // constructor elided...

    public void Run()
    {
        _handler();
    }
}
```

I have yet to explore this style in-depth myself, but it seems promising. (I would avoid using pure `Func<T, ...>` as it doesn’t give any indication of what the intention of the code is.)

The nice thing about delegates is that they will implicitly convert any compatible lambda; so if you need a one-off implementation in test code, you can write it directly in your test method, and not have to create an entirely new class.
