---
layout: post
published: true
title: Improving the carnac codebase and Rx usage (Part 3)
comments: true
---

Parts one and two have covered the InterceptKeys class which exposes a feed of raw keyboard events and the MessageProvider which turns key presses into messages to display on the screen. There is a class between those two, the KeyProvider which I am not going to actually blog about as it doesn't introduce any new concepts which I have not covered in parts 1 and 2 of this blog series.

Instead we will dive into the MessageController class which brings the streams from the KeyProvider and MessageProvider together then maintains the list of Messages displayed on the screen. The requirements are:

 - Add any new messages into the messages collection
 - After 5 seconds set a flag on the message which triggers an animation
 - 1 second after that the message is removed from the messages collection
 - If a message is updated the 5 second countdown should start again.

## Old implementation
The previous implementation was quite simple but it was hard to test. Also Rx can solve this problem very nicely, so why not.

 - New messages were added to the messages list
 - A timer fired once a second, the callback did:
   - A foreach loop over every item which had a LastUpdated property more than 5 seconds go, then set the IsDeleting flag to true
   - A foreach loop over every item which had a LastUpdated property more than 6 seconds go, then removed it from the list

Lets rewrite this into Rx

# Implementing in Rx
## Messages stream
``` csharp
IObservable<KeyPress> keyStream = keyProvider.GetKeyStream();
IObservable<Message> messageStream = messageProvider.GetMessageStream(keyStream);
```

Step one is to get the messages subscription. Nothing much to see here except the variable names and types.

## Adding messages
``` csharp
var addMessageSubscription = messageStream.Subscribe(message => messages.Add(message));
```

Also not much to see here, we subscribe to the messages stream and when a new value is yeilded we add that into the message into the messages collection.

## Fading messages out
Now is where things become a little more interesting. Lets start off with simply setting the flag after 5 seconds

```csharp
var fadeOutMessageSubscription = messageStream
    .Delay(FiveSeconds)
    .Subscribe(m => m.IsDeleting = true);

```

Great, but this does not take into account the updates. Here is the problem we want to solve visualised.

Each - represents a second, | is a completed stream

```
a--------b---------a----*ab----
-----a|
         -----b|
                   ---------ab|
-----a--------b-------------ab
```

This shows us that we need a nested stream of some sort. Lets rewrite the fade out query into something that will create an inner stream.

``` csharp
var fadeOutMessageStream = messageStream
    .SelectMany(message =>
    {
        // the inner stream goes here
    });
var fadeOutMessageSubscription = fadeOutMessageStream.Subscribe(m => m.IsDeleting = true);
```

Now we need to write the inner stream. Once again lets visualise the problem (x is an update, @ is the start of an observable.Timer(), o is the stream yeilding a value)

```
x---x----x-----
@---|
    @----|
         @-----o|
---------------x|
```

The thing that is changing and we are reacting to in this query is the updates of the message. A message has a property with signature `IObservable<Unit> Updates { get; }` which we can subscribe to for updates to that message. We also want to kick off right away when we subscribe (otherwise the timer will not start until an update happens). With this in mind, the start of the inner query is:

``` csharp
return message.Updated
    .StartWith(Unit.Default);
```

Now we need to introduce a timer of some sort. Because each update has it's own timeout we need another nested stream. The marble diagram shows the nested screen pattern off really nicely. To get the nested stream we just do a `.Select()` which give us:

``` csharp
return message.Updated
    .StartWith(Unit.Default)
    .Select(_ => Observable.Timer(FiveSeconds))
```

The return type of that select is actually `IObservable<IObservable<long>>` which is the nested stream. In the marble diagram above you will notice that when we get an update and start a timer we close the previous timer stream. In Rx this is known as the `.Switch()` statement. When the outer observable yeilds a new inner observable, switch will dispose the last inner observable and subscribe to the new stream instead. Now the code is:

``` csharp
return message.Updated
    .StartWith(Unit.Default)
    .Select(_ => Observable.Timer(FiveSeconds))
    .Switch();
```

We are not finished yet, switch returns us a single stream of type `IObservable<long>`. We need a stream of messages again:

``` csharp
var fadeOutMessageStream = messageStream
    .SelectMany(message =>
    {
        return message.Updated
            .StartWith(Unit.Default)
            .Select(_ => Observable.Timer(FiveSeconds))
            .Switch()
            .Select(_ => message);
    });
```

We now are back showing the code which includes the outer stream. Our select we have just added simply ignores what it gets given and returns the message passed by the outer query.

Going back to our marble diagram you will notice there is one thing missing:

```
a--------b---------a----*ab----
-----a|
         -----b|
                   ---------ab|
-----a--------b-------------ab
```

After each inner stream yeilds a value, it immediately terminates. This is an easy fix, we just append `.Take(1)` to the inner query giving us:

``` csharp
var fadeOutMessageStream = messageStream
    .SelectMany(message =>
    {
        return message.Updated
            .StartWith(Unit.Default)
            .Select(_ => Observable.Timer(FiveSeconds))
            .Switch()
            .Select(_ => message)
            .Take(1);
    });

var fadeOutMessageSubscription = fadeOutMessageStream.Subscribe(m => m.IsDeleting = true);
```

Onto the [Next song](https://www.youtube.com/watch?v=Rre3zgL7eMk#t=56)

## Removing messages
The nice thing about Rx is that you can just build on other streams. So we just take our `fadeOutMessageStream`, apply a 1 second delay then subscribe.

``` csharp
var removeMessageStream = fadeOutMessageStream
    .Delay(OneSecond)
    .Subscribe(m => messages.Remove(m));
```

## Next steps
One thing all of these streams rely on is that each of them receive the *same instance* of each message. It is no good to us if each of them are working on a different messages. This is an easy fix using an operator we saw in Part 1 of this series, `.Publish()`.

But this time we will not use `.RefCount()` as we want to set up our queries then turn them on all at the same time, otherwise we have race conditions that messages might be added but never removed. We want all 3 of our subscriptions to start simultanously.

Lets zoon out to the method level and have a look what the `KeyController.Start` method looks like after we publish the stream.

``` csharp
public IDisposable Start()
{
    var keyStream = keyProvider.GetKeyStream();
    IConnectableObservable<Message> messageStream = messageProvider.GetMessageStream(keyStream).Publish();

    IDisposable addMessageSubscription = ...;
    IDisposable fadeOutMessageSubscription = ...;
    IDisposable removeMessageStream = ...;

    IDisposable underlyingMessageSubscription = messageStream.Connect();

    return new CompositeDisposable(
        underlyingMessageSubscription,
        addMessageSubscription,
        fadeOutMessageSubscription,
        removeMessageStream);
}
```

We simple return a new `CompositeDisposable` which takes any number of IDisposable's as constructor parameters, then when it is disposed it will dispose all of the things passed to it.

To walk through what we see above, I publish the message stream, create the three subscriptions and then I call `.Connect()` on the messageStream (which is now of type `IConnectableObservable<Message>`).

This allows me to explicitly control the underlying stream, when disposing I simply dispose the underlying stream then all of my other streams and everything is cleaned up nicely.

### Specifying Schedulers
One part of the code I have ommited is explicitly specifying on what Scheduler everything will run on. I have introduced an `IConcurrencyService` which abstracts me from the static Rx schedulers (allowing me to write tests, which is coming up next!). When using Rx you should always specify what schedulers you want your code to run on, in carnac's case we want everything to run on the UI Thread. Normally this is bad but because we are modifying messages which are bound to by the UI we have to. Maybe a future blog post can be how we made carnacs Rx streams immutable, but not now.

We do that by adding these two lines before every `.Subscribe()` call in this class.

``` csharp
.ObserveOn(concurrencyService.MainThreadScheduler)
.SubscribeOn(concurrencyService.MainThreadScheduler)
```

### Error handling
I wanted to call our error handling explicitly, carnac has *none*. It didn't before this refactor and it didn't after. There is a GitHub issue open at the time of writing this blog post to sort this out. 

## Testing
I mentioned at the start of this post that this would be easier to test, lets put that to the test and write some tests. First off the simplest case.

``` csharp
[Fact]
public void MessagesAreAddedIntoKeysColletion()
{
    var message = new Message(A);

    sut.Start();
    ProvideMessage(message);

    keysCollection.ShouldContain(message);
    message.IsDeleting.ShouldBe(false);
}
```

A is a property which returns a KeyPress and ProvideMessage simply yields that message to the messageStream which the KeyProvider subscribes to. Now we have an simple example test, here is a more complex example. 

``` csharp
[Fact]
public void MessagesAreFlaggedAsDeletingAfter5Seconds()
{
    var message = new Message(A);

    sut.Start();
    ProvideMessage(message);
    testScheduler.AdvanceBy(TimeSpan.FromSeconds(5).Ticks);

    message.IsDeleting.ShouldBe(true);
}
```

In this test we have made use of our ICurrencyService by making it always return a `TestScheduler`. The `TestScheduler` is in the Rx.Testing NuGet library and is **very** useful. Because Rx abstracts time we can write tests which rely on time very quickly and they also execute very fast. 

The above test provides a message to the messageStream and then moves time forward by 5 seconds and makes sure that our message has been flagged for deletion, which it has. Lets push this a little harder and make sure that we do not expire when that message has been updated.

``` csharp
[Fact]
public void MessageTimeoutIsStartedAgainIfMessageIsUpdated()
{
    var message = new Message(A);

    sut.Start();
    ProvideMessage(message);
    testScheduler.AdvanceBy(TimeSpan.FromSeconds(3).Ticks);
    message.Merge(new Message(A)); // Update message
    testScheduler.AdvanceBy(TimeSpan.FromSeconds(3).Ticks);

    message.IsDeleting.ShouldBe(false);
    keysCollection.ShouldContain(message);
}
```

As you can see the code to test our Rx queries is really simply. 

# Summary
The MessageController is quite a small class and reasonably simple once you are familiar with Rx concepts but it has some interesting problems which many applications have and demonstrates how they can be easily solved and tested.

This concludes this series and I hope it has been useful to you. I have wanted to do this refactoring for quite a while as carnac is a great sample Rx application as the problem is inherently stream based. Before this refactor though there were many bad practices being used. I am sure that things could be improved upon more but I leave that up to you. If you see a way the carnac codebase could be cleaned up or improved please jump in and submit a pull request. 