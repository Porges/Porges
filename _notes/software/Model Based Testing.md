---
title: Cheap model-based testing with F# & FsCheck
date: 2017-02-01
---
_This is a blog version of a talk that I did ~2015 in Auckland. Nothing in it is original or unknown, but there aren’t that many examples of doing this aside from [in the FsCheck documentation](https://fscheck.github.io/FsCheck/StatefulTesting.html). I also gave [a talk which contains a very similar example](https://www.youtube.com/watch?v=8oALNLdyOyM) to the Melbourne ALT.NET meetup in 2018._
### What is FsCheck?

FsCheck is a library in the spirit of Haskell’s [QuickCheck](https://en.wikipedia.org/wiki/QuickCheck), which allows you to perform property-based testing. The main feature that it provides is the ability to generate arbitrary instances of any supported data type, and the ability to define your own generators.

We can leverage this ability to perform model-based testing in a simple manner.

Let’s run through how we can use this to test a simple system. All the code in this post is going to be F#, but the system could equally well be described in C# and tested from F#. If you copy and paste the code into an F# file, it should work (you’ll need to install the `FsCheck.Xunit` and `xunit` packages, and probably `xunit.runner.visualstudio` to run the tests).

### The system to test

In our basic system, we store users. We can `Add`, `Get` (possibly failing), or `Delete` a user. This is described by the following interface:

```fsharp
[<Struct>]
type UserId = UserId of string

type User = { id : UserId ; name : string ; age : int }

type IUserOperations =
  abstract member AddUser : User -> unit
  abstract member GetUser : UserId -> Option<User>
  abstract member DeleteUser : UserId -> unit
```

Next we have to implement this system.

We’ll use an in-memory dictionary, where we map `UserID`s to the actual `User` objects:

```fsharp
open System.Collections.Generic

type UserSystem() = 
  let users = Dictionary<UserId, User>()

  // Exposed for us to use later:
  member this.UserCount = users.Count

  // And implement the interface:
  interface IUserOperations with
    member this.AddUser u =
        users.[u.id] <- u

    member this.GetUser id =
      match users.TryGetValue id with
      | (true, user) -> Some user
      | _ -> None

    member this.DeleteUser (UserId id) =
      if id.Contains "*" // Uh-oh: catastrophic bug!
      then users.Clear()
      else users.Remove (UserId id) |> ignore
```

_Unfortunately_, on line 20 we’ve introduced a critical bug. If the user ID contains “`*`” then we accidentally remove all the users in the system. Oops! 🤦

Later on we will see if our test will find this bug.

### Modelling the system

In order to check our system, we must develop a model—a simplified version of the system that we can verify more easily. The model doesn’t need to replicate every piece of behaviour of the full system, it might only replicate one aspect that we care about.

Here we will use a version of the system that only cares about _how many_ users there are. In order to track this it only needs to store the set of valid IDs:

```fsharp
// this is our model
// the only thing we care about is how many users there are
// so we just store the names as a set
type UserCountModel() =

    let users = HashSet<UserId>()

    member this.Verify (real : UserSystem) =
        Xunit.Assert.Equal(users.Count, real.UserCount)
        
    interface IUserOperations with
        member this.AddUser u = 
            users.Add u.id |> ignore

        member this.DeleteUser name =
            users.Remove name |> ignore

        member this.GetUser name = None
```

`Add` adds an ID to the set (unless it already exists), `Delete` removes an ID (if it exists), and `Get` does nothing.

We also provide a `Verify` method that checks the model against the real system. (In real life, this could use HTTP calls or some other method.)

### Applying operations

Now, in order to test our system, we need a way to describe the actions we can perform against it. We do this by _reifying_ the operations we can perform as a data type:

```fsharp
type Operation =
    | Add of User
    | Get of UserId
    | Delete of UserId
```

See that that this aligns exactly with the `IUserOperations` interface shown in the first block of code above. For each method on the interface, we have a corresponding case in the discriminated union, and the data needed to call the method is also contained in the case.

Next, we need to be able to perform these actions against both the model and the real system. To do this we simply translate from each case to the corresponding method on the interface:

```fsharp
let applyOp (op : Operation) (handler : IUserOperations) =
    match op with
    | Add user -> handler.AddUser user
    | Get name -> handler.GetUser name |> ignore
    | Delete name -> handler.DeleteUser name
```

### Testing the system

Now we get to the magic part! First up, we have a little boilerplate to tell FsCheck that we don’t want any `null` strings generated, as this would just complicate the example:

```fsharp
open FsCheck
open FsCheck.Xunit

type Arbs = 
    // declare we only want non-null strings
    static member strings() =
        Arb.filter (fun s -> s <> null) <| Arb.Default.String()

[<Properties(Arbitrary=[|typeof<Arbs>|])>]
module Tests =
  // ... next snippet
```

And now our test. We take in a list of operations (which FsCheck will automatically generate for us), we apply all the operations to both models, and then we verify the system against the model:

```fsharp
    [<Property>]
    let ``Check implementation against model`` operations =

        // create both implementations empty
        let real = UserSystem()
        let model = UserCountModel()

        // apply all the operations
        let applyToBothModels op =
            applyOp op real
            applyOp op model

        List.iter applyToBothModels operations

        // verify the result
        model.Verify real
```

This may seem like less code than expected! FsCheck can automatically build us a list of `Operation`s because it knows how to make `Operation` (since it knows how to make each case in turn,starting from `string`s and building up). Recursion is great!

### Running the test and “shrinking”

If you try to run the test, you should receive a result along these lines (the exact details will differ):

> TODO: the image has gone dead

Our test failed (that’s a good thing!) See the second-to-last `Delete` operation in the input? It is deleting `UserId "^*VJc6'/I^x"`, which triggers the bug we have in our system for user IDs containing “`*`”.

The input it’s showing us is quite gnarly as FsCheck has generated user IDs containing newlines and other nasty characters, but the highlighted part at the bottom is what FsCheck was able to shrink the failing input to.

It can do this because, as well as knowing how to _generate_ random data, FsCheck knows how to _shrink_ random data. It was able to repeatedly shrink the input until it found this minimal case, containing only two operations:

1. Add a user (note that it doesn’t seem to matter what the details are, as all the attributes shrank away to nothing–if the details mattered, then it wouldn’t have shrunk this much)
2. Delete the user with ID “`*`”

And indeed, this sequence exactly replicates our bug! 🎉

The combination of randomly generating inputs and then being able to shrink them when failures are found is _very_ powerful.

### The recipe

A simple list for success:

1. model your system-under-test as an interface
2. create a model of your system (or multiple models, each testing one aspect) that also implements the interface
3. create a type that reifies each operation as a member of a discriminated union
4. create a test that applies a list of operations to the model and system and validates the system
5. 💅 relax and let your computer do the work
