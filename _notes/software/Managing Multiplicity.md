---
date: 2000-01-03
---
_Alternate titles: Purging Pluralities, Collapsing Cardinality._

Dealing with zero-or-more implementations is a common occurrence when dealing with code that uses behaviour injection (AKA the strategy pattern) heavily.

In this post I’ll demonstrate how to deal with multiplicity for _specific types_ of strategies, in order to remove complexity from the consumer and allow more reuse and flexibility.

Essentially, we’ll discover how to _mechanize_ the production of [composites](http://c2.com/cgi/wiki?CompositePattern), for types that have some form of _monoidal_ result (which I’ll define at the end).

## The scenario

If we have a class that has some behaviour that is injected, it often makes sense to generalize to allow multiple instances of that behaviour to be injected. We might end up with code that looks something like this:

```csharp
interface IStrategy
{
    Result DoTheThing(Input input);
}

class Consumer
{
    readonly ICollection<IStrategy> _strategies;

    public void PerformDuties()
    {
        foreach (var strategy in _strategies)
        {
            var result = strategy.DoTheThing(/*...*/);
            // do something with result
        }
    }
}
```

This works fine. But there are a few reasons we should avoid this construct.

The simplest is that we’ll probably end up duplicating this code if we need to use the `IStrategy` in several places – once one consumer accepts more than one instance, other consumers are likely to need to as well.

A deeper problem is that the interpretation of multiple instances—that is, _how the instances are combined_—is determined by the _consumer_ class and not by the _strategy_. It would be more coherent for the `IStrategy` interface to decide how to combine multiple instances.

It turns out that there’s a simple way to extract this complexity, and allow the strategies to decide how to combine themselves … for certain types of strategy. Thankfully these ‘certain strategies’ are some of the most common ones. In this post I’ll show how this works for:

- strategies that decide _whether or not_ to perform some other action (“predicates” or “policies”)
- strategies that are used just for their behaviour and not their result (“actions” or “handlers”, fire-and-forget strategies)
- strategies where you want to collect the results into a list (“samplers”?)

Let’s examine how to deal with each of these, and then we’ll see how to generalize this to other types of strategies.

### Combining policies/predicates

A _policy_ decides whether or not to perform an action. Usually when dealing with multiple policies you only want to perform the action if _all_ of them permit it.

In code, this will look something like this. We have an `IPolicy` interface and a consumer class:

```csharp
interface IPolicy
{
    bool ShouldPerformAction(Item item);
}

class UsesPolicy
{
    readonly ICollection<IPolicy> _policies;

    public void PerformDuties(Item item)
    {
        foreach (var policy in _policies)
        {
            if (!policy.ShouldPerformAction(item))
            {
                return;
            }
        }

        // we got here, so perform the action
    }
}
```

If any of the policies return `false`, we’ll bail out of the method, but otherwise we will perform the action.

In order to reduce the complexity of the consumer class, we can create a new implementation of the `IPolicy` interface, which _knows how to combine other policies_:

```csharp
class CombinedPolicy : IPolicy
{
    [NotNull]
    readonly IReadOnlyCollection<IPolicy> _policies;

    public CombinedPolicy(IEnumerable<IPolicy> policies)
    {
        _policies = policies.ToList();
    }

    public bool ShouldPerformAction(Item item)
        => _policies.All(p => p.ShouldPerformAction(item));
}
```

Note that `ShouldPerformAction` is (functionally) equivalent to what the consumer class was doing before.

Given this class, we have a _generic way to combine any number of policies into a single policy_. Note also that `CombinedPolicy` correctly deals with the case where we have _no_ policies (it will return `true`)—this will be important later.

The end result is that our consumer class will only ever need to deal with _one_ policy at a time!

So, let’s simplify it:

```csharp
class UsesPolicyRevised
{
    readonly IPolicy _policy;

    public void PerformDuties(Item item)
    {
        if (_policy.ShouldPerformAction(item))
        {
            // perform the action
        }
    }
}
```

Great! Now we only have to deal with one `IPolicy` at a time. The added bonus is that if we have any other classes that use `IPolicy`, they will now _automatically_ be able to deal with _any_ number of policies, and they way they deal with multiple policies will be consistent. 👍

### Combining fire-and-forget strategies

This situation might arise if you want to send a message to several different destinations. Maybe you want to log a message, but also send it to an alerting system.

Let’s start with a similar setup to the previous one. We have an `IDoStuff` action interface and a consumer:

```csharp
interface IDoStuff
{
    void DoStuff(Item item);
}

class InvokesActions
{
    readonly ICollection<IDoStuff> _actions;

    public void PerformDuties(Item item)
    {
        foreach (var action in _actions)
        {
            action.DoStuff(item);
        }
    }
}
```

Here we have a fan-out situation where multiple actions are called on the same value.

If you’ve been following closely, you’ll know what we have to do next: create a class that does this for us. The important thing is that it is also an instance of `IDoStuff`:

```csharp
class CombinedAction : IDoStuff
{
    [NotNull]
    readonly IReadOnlyCollection<IDoStuff> _actions;

    public CombinedAction(IEnumerable<IDoStuff> actions)
    {
        _actions = actions.ToList();
    }

    public void DoStuff(Item item)
    {
        foreach (var action in _actions)
        {
            action.DoStuff(item);
        }
    }
}
```

Now our original class becomes simply:

```csharp
class InvokesActionsRevised
{
    readonly IDoStuff _action;

    public void PerformDuties(Item item)
    {
        _action.DoStuff(item);
    }
}
```

That was pretty easy!

Something else important has happened here: not only can we give the consumer class multiple actions, but we can customize _how multiple actions are performed_.

Let’s change the interfaces a bit to make it more interesting. Imagine instead that the `IDoStuff` interface was asynchronous:

```csharp
interface IDoStuffAsync
{
    Task DoStuff(Item item, CancellationToken ct);
}
```

Now there are two plausible implementations of `CombinedAction`. One will run the actions in sequence, like before (I won’t show this here), but another possibility is that we want to fire off all the actions at once, in parallel:

```csharp
class InParallelActions : IDoStuffAsync
{
    [NotNull]
    readonly IReadOnlyCollection<IDoStuffAsync> _actions;

    public InParallelActions(IEnumerable<IDoStuffAsync> actions)
    {
        _actions = actions.ToList();
    }

    public Task DoStuff(Item item, CancellationToken ct)
        => Task.WhenAll(_actions.Select(action => action.DoStuff(item, ct)));
}
```

This gives us more flexibility, and we can make decisions about how the actions are to be performed (in sequence or in parallel) _without changing the consuming class_.

### Collecting results into a list

This situation might arise if you have a range of ‘samplers’ that you want to run on some object. Perhaps you want to read several different indicators of “health” from a computer—performance counters, IO stats, memory usage and so on—and then collect these all these results into a list to store for later retrieval.

Each individual collector would be a separate class, so that you can develop & test them in isolation.

The initial setup should be familiar by now. We have an interface `ICollectInformation` and a consumer class:

```csharp
struct Datum
{
    public string Key { get; }

    public double Value { get; }
}

interface ICollectInformation
{
    Datum CollectInformation();
}

class CollectsInformation
{
    readonly ICollection<ICollectInformation> _collecters;

    public void DoCollection()
    {
        var data = new List<Datum>();

        foreach (var collector in _collecters)
        {
            data.Add(collector.CollectInformation());
        }
        
        // do something with data
    }
}
```

At this point, we want to be able to create an instance which will collect items into a list for us. However, we can’t return a list from `ICollectInformation`—it can only return a single `Datum`. So, let’s alter this interface (it’s our code, after all), and change it to return any number of items:

```csharp
interface ICollectInformation
{
    IEnumerable<Datum> CollectInformation();
}
```

Now we can write the implementation of `ICollectInformation` that we want:

```csharp
class AggregateCollector : ICollectInformation
{
    [NotNull]
    readonly IReadOnlyCollection<ICollectInformation> _collectors;

    public AggregateCollector(IEnumerable<ICollectInformation> collectors)
    {
        _collectors = collectors.ToList();
    }

    public IEnumerable<Datum> CollectInformation()
        => from collector in _collectors
           from datum in collector.CollectInformation()
           select datum;
}
```

And our consumer class again only has to deal with a single instance:

```csharp
class CollectsInformationRevised
{
    readonly ICollectInformation _collecter;

    public void Collect()
    {
        var data = _collecter.CollectInformation().ToList();
        
        // do something with data
    }
}
```

Once again, complexity averted!

## The general case

So, what’s the magic ingredient here that allows us to combine these types of strategies? The key is that the result type of each can be treated as a _monoid_.

A type is a **monoid** (or is **monoidal**), if it satisfies three conditions:

> 1. You can combine two (or more) instances of it to form something that has the same type.

This allows us to “squash” multiple return values together, so we can reduce many strategies to one strategy.

> 2. The combining function (call it `#`) must be associative so that it doesn’t matter “which way” you group the instances when combining—`x # (y # z)` should be the same as `(x # y) # z`. (But order can matter, `x # y` doesn’t have to be the same as `y # x`.)

Although it wasn’t used above, this means we can nest strategies arbitrarily using our combiners, and it won’t affect the result.

> 3. There is an instance (called the “identity”) that can be combined with any other instance without changing the result.

This is important as it allows us to produce a result in the case that we have zero strategies, and gives us sane results in other cases.

Here’s how the previous examples match up to the monoid concept:

In the first example (combining policies), we were (1.) combining multiple booleans using `&&` under the covers to get back another boolean. For condition (2.), it doesn’t matter if you write `a && (b && c)` or `(a && b) && c`, as it will give you the same answer, and (3.) if you combine `true` with any boolean using `&&`, you get back the original boolean, so `true` is the identity for `&&`.

For the second example (combining actions) we were essentially using `void` as a monoidal type. It’s not very interesting since combining two `void`s (via sequencing) just gives you another `void`—so the third condition is trivially satisfied.

For the third example (combining samplers), we were using `IEnumerable`. In this case, the combining operation is concatenation, and the identity is the empty enumerable. Since a monoid combining operation must produce something of the same type as its inputs, and we can’t combine two `Datum`s to make another `Datum`, we had to change the result type of the `ICollectInformation` interface to return an enumerable before we could create the combining class.

I tried to pick three monoids that are useful for dealing with strategies here. Numbers can also be monoids ([the sum or product monoids, or max or min monoids](https://en.wikipedia.org/wiki/Monoid#Examples)), but I don’t think they’re as useful in this particular situation.

So, to summarize: Once you have discovered that the result type of a strategy can be treated as a monoid, then you can apply the above techniques to result in simpler, cleaner code:

1. Identify what the combining operation for a particular strategy type is.
2. Extract a class that performs that operation (and is an instance of the same interface).
3. Remove the complexities of the combining operation from the consuming classes, so that they only need to consider a single instance.
#### Extra: Making combining more palatable (C#-specific)

If possible, you could also switch out the interface for an abstract base class. This allows us to define operators on the class, so we can easily combine multiple instances without having to use the ‘combiner’ type explicitly. Indeed, we can hide the combiner type internally:

```csharp
abstract class PolicyType
{
    public abstract bool ShouldPerformAction(Item item);
    
    public static PolicyType operator&(PolicyType first, PolicyType second)
        => new CombinedPolicy(first, second);
    
    // this could be much cleaner if C# ever gets F#-style object expressions:
    // https://github.com/dotnet/roslyn/issues/13#issuecomment-195161037
    private class CombinedPolicy : PolicyType
    {
        readonly PolicyType _first;
        readonly PolicyType _second;

        public CombinedPolicy(PolicyType first, PolicyType second)
        {
            _first = first;
            _second = second;
        }

        public override bool ShouldPerformAction(Item item)
            => _first.ShouldPerformAction(item) && _second.ShouldPerformAction(item);
    }
}
```

Then we can simply combine policies via `p1 & p2`. Note that this relies on the associativity of the monoid to work well!

#### A final note

The deeper reason that this all works is not only that the _result_ type is monoidal, but that `Func<TArg, TResult>` is _itself_ a monoid, if its result is a monoidal type. This allows us to combine multiple instances of `Func<TArg, TResult>`. See if you can figure out what the combining function is for it! I’ll also explore this more in a future post.