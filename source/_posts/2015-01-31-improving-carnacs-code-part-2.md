---
layout: post
published: true
title: Improving the carnac codebase and Rx usage (Part 2)
comments: true
---

This post is the second part in a series covering a series of improvements in the carnac codebase, specifically to improve the usage of Rx. The next class I will be rewriting is the MessageProvider.

As a bit of background, in carnac a KeyPress is not directional and it also contains information about if modifiers were pressed at the same time. For Instance `ctrl + r` would be a KeyPress. A `Message` is what is shown on the screen.

The message provider as it is does the following:

 - Is IObserver<KeyPress>
 - It aggregates multiple KeyPresses into logical messages with the following rules:
   - Shortcuts are always shown in their own message
   - If there has been more than a second between the last keypress a new message is created
   - If the key presses were entered into different applications a new messsage is created
 - Apply 'Only show shortcuts' filter

Here is what the code looked like before the refactor.

``` csharp
public class MessageProvider : IMessageProvider, IObserver<KeyPress>
{
    readonly Subject<Message> subject = new Subject<Message>();
    private readonly IShortcutProvider shortcutProvider;
    private IDisposable keyStream;
    private readonly PopupSettings settings;

    public MessageProvider(IKeyProvider keyProvider, IShortcutProvider shortcutProvider, ISettingsProvider settingsProvider)
    {
        this.shortcutProvider = shortcutProvider;
        settings = settingsProvider.GetSettings<PopupSettings>();
        keyStream = keyProvider.Subscribe(this);
    }

    public Message CurrentMessage { get; private set; }

    public IDisposable Subscribe(IObserver<Message> observer)
    {
        return subject.Subscribe(observer);
    }

    public void OnNext(KeyPress value)
    {
        Message message;

        var currentKeyPress = new[] {value};
        var keyPresses = CurrentMessage == null ? currentKeyPress : CurrentMessage.Keys.Concat(currentKeyPress).ToArray();
        var possibleShortcuts = GetPossibleShortcuts(keyPresses).ToList();
        if (possibleShortcuts.Any())
        {
            var shortcut = possibleShortcuts.FirstOrDefault(s => s.IsMatch(keyPresses));
            if (shortcut != null)
            {
                message = CurrentMessage ?? CreateNewMessage(value);
                message.AddKey(value);
                message.ShortcutName = shortcut.Name;
                //Have duplicated as it was easier for now, this should be cleaned up
                return;
            }
        }

        // Haven't matched a Chord, try just the last keypress
        var keyShortcuts = GetPossibleShortcuts(currentKeyPress).ToList();
        if (keyShortcuts.Any())
        {
            var shortcut = keyShortcuts.FirstOrDefault(s => s.IsMatch(currentKeyPress));
            if (shortcut != null)
            {
                //For matching last keypress, we want a new message
                message = CreateNewMessage(value);
                message.AddKey(value);
                message.ShortcutName = shortcut.Name;
                //Have duplicated as it was easier for now, this should be cleaned up
                return;
            }
        }

        if (!value.IsShortcut && settings.DetectShortcutsOnly)
            return;
        
        if (ShouldCreateNewMessage(value))
        {
            message = CreateNewMessage(value);
        }
        else
            message = CurrentMessage ?? CreateNewMessage(value);

        message.AddKey(value);
    }

    private Message CreateNewMessage(KeyPress value)
    {
        var message = new Message
                              {
                                  StartingTime = DateTime.Now,
                                  ProcessName = value.Process.ProcessName
                              };

        CurrentMessage = message;
        subject.OnNext(message);
        return message;
    }

    private bool ShouldCreateNewMessage(KeyPress value)
    {
        return
            CurrentMessage == null ||
            IsDifferentProcess(value) ||
            IsOlderThanOneSecond() ||
            LastKeyPressWasShortcut() ||
            value.IsShortcut;
    }

    private bool LastKeyPressWasShortcut()
    {
        return CurrentMessage.Keys.Last().IsShortcut;
    }

    private IEnumerable<KeyShortcut> GetPossibleShortcuts(IEnumerable<KeyPress> keyPresses)
    {
        return shortcutProvider.GetShortcutsMatching(keyPresses);
    }

    private bool IsOlderThanOneSecond()
    {
        return CurrentMessage.LastMessage < DateTime.Now.AddSeconds(-1);
    }

    private bool IsDifferentProcess(KeyPress value)
    {
        return CurrentMessage.ProcessName != value.Process.ProcessName;
    }

    public void OnError(Exception error)
    {

    }

    public void OnCompleted()
    {

    }
}
```

The first thing you will notice is that most of the methods in this class access and manipulate the property CurrentMessage. This makes this class really hard to follow and rationalise what is going on. 

# Removing IObservable<T>
Like in the previous class we refactored, the first thing we need to do is to stop implementing IObservable<T>. 

Our subscribe method will change from `IDisposable Subscribe(IObserver<Message> observer)` to `IObservable<Message> GetMessageStream(IObservable<KeyPress> keyStream)`

This means consumers of this class can simply pass an observable in and get a new feed. It also makes this class easy to test. 

# Visualising the requirements
After looking at the current behaviour I decided that it would make more sense to not show a partial shortcut on the screen until it had either been completed or broken. With that in mind, we will start off with a series of key presses. a, b, ctrl+r, ctrl+r, ctrl+r, a, ↓, ↓ (↓ is the down arrow key). For these examples imagine we have a single shortcut which is ctrl+r, ctrl+r.

If we draw an ascii marble diagram it will look like this

`a----b----ctrl+r----ctrl+r----ctrl+r----a----↓----↓`

The first requirement is that we batch shortcuts into a single message. 

```
a----b----ctrl+r----ctrl+r----------ctrl+r----a-----------↓---↓
a----b--------------ctrl+r,ctrl+r-------------ctrl+r--a---↓---↓
```

In the above diagram we see that ctrl+r, ctrl+r is a completed shortcut so our second stream emits the completed shortcut. But when we have ctrl+r, 'a' that is not a shortcut. It is instead a broken shortcut so the second stream emits two messages, one directly after the other. a, b and the arrow keys emit a completed message right away because they are not part of any potential shortcuts.

The next step is to merge messages together which we want to display on the screen together. In the above example we want to see this on the screen:

```
ab
ctrl+r, ctrl+r [Rename]
crtl+r
a↓ x 2
```

Lets add that into our marble diagram. Items prefixed with * are new messages which will replace the messages it has been merged with
```
a----b----ctrl+r----ctrl+r----------ctrl+r----a-----------↓----↓-------
a----b--------------ctrl+r,ctrl+r-------------ctrl+r--a---↓----↓-------
a----*ab------------ctrl+r,ctrl+r-------------ctrl+r--a---*a↓--*ab↓ x 2
```

Now we have an idea visually of what our streams will look like we can turn this into Rx.

# Writing the query
In Rx when we want to reduce the number of items we have where we need some sort of aggregation function. In this case we want to use the `Scan` operator.

``` csharp
return keyStream
    .Scan(new ShortcutAccumulator(), (acc, key) => acc.ProcessKey(shortcutProvider, key))
    .Where(c => c.HasCompletedValue)
    .SelectMany(acc => acc.GetMessages())
```

Scan calls your accumulation function for each item in the stream, but unlike `Aggregate` it will emit the new aggregated value. In this case we start off with an empty ShortcutAccumulator, when we get a key press we simply pass it to the accumulator along with the shortcut provider.

The ShortcutAccumulator will check if that key press matches any shortcuts. If it doesn't, or that key completes the shortcut, it sets the HasCompletedValue property to true and returns itself. When a completed accumulator is asked to process a key it will simply create a new ShortcutAccumulator and get it to process the key and return that accumulator instead of itself.

Because Scan will emit the ShortcutAccumulator after each key press, we can simply filter the accumulators which are not completed yet, then select many on each completed ShortcutAccumulator to get the messages it has accumulated. The reason for the select many is when a shortcut is broken we create a message for each accumulated key press. And our stream now matches the second line in our marble diagram.

To do the final line in our marble diagram we need another `Scan` which merges Messages which need to be merged.

``` csharp
return keyStream
    .Scan(new ShortcutAccumulator(), (acc, key) => acc.ProcessKey(shortcutProvider, key))
    .Where(c => c.HasCompletedValue)
    .SelectMany(c => c.GetMessages())
    .Scan(new Message(), (previousMessage, newMessage) => messageMerger.MergeIfNeeded(previousMessage, newMessage))
    .Where(m => !settings.DetectShortcutsOnly || m.IsShortcut);
```

Whether a message needs to be merged or not is no longer the responsibility of this class, it has been moved into our MessageMerger class. MergeIfNeeded will check for all the merge conditions like the key presses being over a second apart, from different processes, either message being a shortcut etc and if they can be merged the new message will be merged into the previous. Ideally our entire stream would be immutable but that will have to be a separate task, we are refactoring existing code after all. 

Finally we apply the Shortcut Only setting and filter our list if that option is set.

# Summary
The end result of the MessageProvider refactoring was a great improvement. The end result is a single method returning an Rx statement which forfills all of our requirements. The shortcut detection logic was moved into another class which has a single responsibility making it much easier to understand what is going on and follow. 

