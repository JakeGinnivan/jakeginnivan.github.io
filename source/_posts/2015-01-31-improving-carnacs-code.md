---
layout: post
published: true
title: Improving the carnac codebase and Rx usage (Part 1)
comments: true
---

[Carnac](http://carnackeys.com/) is an open source project created as part of [Code52](http://code52.org/). It is a simple utility which overlays key presses on your screen as you type which is pretty handy for presentations. Carnac also ships with some keymaps for different applications so it understands when you are pressing shortcuts.

Recently Scott Hanselman blogged about a [Quake Mode console](http://www.hanselman.com/blog/QuakeModeConsoleForVisualStudioOpenACommandPromptWithAHotkey.aspx) and mentioned carnac, this triggered Brendan Forster and myself to think about carnac again. When we started writing carnac there was not any Rx experience amongst the team and we did a pretty bad job. I think carnac is a *great* Rx problem and the code can be improved to take advantage of Rx and be a really good non-trivial Rx sample.
This post is all about the things we did wrong and the process I went through to refactor carnac into a much simpler code base and really leverage the power of Rx.

# Fixing the InterceptKeys class
Carnac uses windows low level keyboard hooks to listen to key pressess. We have a class called `InterceptKeys` which is responsible for giving us an Rx stream of KeyEvents. This includes direction so if you press `ctrl+r` the stream would look like this:

 - ctrl (down)
 - r (down)
 - r (up)
 - ctrl (up)

That is all this class has to do. The only other thing to note is `InterceptKeys` is a singleton and should only ever have a single low level keyboard hook created.

Here is the original code

``` csharp
[PermissionSet(SecurityAction.LinkDemand, Name = "FullTrust")]
[PermissionSet(SecurityAction.InheritanceDemand, Name="FullTrust")]
public class InterceptKeys : IObservable<InterceptKeyEventArgs>, IDisposable
{
    static InterceptKeys current = new InterceptKeys();
    readonly Win32Methods.LowLevelKeyboardProc callback;
    readonly Subject<InterceptKeyEventArgs> subject;
    bool disposed;
    IntPtr hookId = IntPtr.Zero;
    decimal subscriberCount;

    InterceptKeys()
    {
        subject = new Subject<InterceptKeyEventArgs>();
        callback = HookCallback;
    }

    public static InterceptKeys Current { return current; }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    public IDisposable Subscribe(IObserver<InterceptKeyEventArgs> observer)
    {
        IDisposable dispose = subject.Subscribe(observer);
        subscriberCount++;
        if (subscriberCount == 1)
            hookId = SetHook(callback);
        return new DelegateDisposable(() =>
                                          {
                                              subscriberCount--;
                                              if (subscriberCount == 0)
                                                  Win32Methods.UnhookWindowsHookEx(hookId);
                                              dispose.Dispose();
                                          });
    }

    IntPtr HookCallback(int nCode, IntPtr wParam, IntPtr lParam)
    {
        if (nCode >= 0)
        {
            /* snip, not really important, just calculating the variabless used below. */

            var interceptKeyEventArgs = new InterceptKeyEventArgs(key, keyDirection, alt, control, shift);

            subject.OnNext(interceptKeyEventArgs);
            if (interceptKeyEventArgs.Handled)
            {
                return (IntPtr)1; //handled
            }
        }

        return Win32Methods.CallNextHookEx(hookId, nCode, wParam, lParam);
    }

    static IntPtr SetHook(Win32Methods.LowLevelKeyboardProc proc)
    {
        using (Process curProcess = Process.GetCurrentProcess())
        using (ProcessModule curModule = curProcess.MainModule)
        {
            return Win32Methods.SetWindowsHookEx(Win32Methods.WH_KEYBOARD_LL, proc,
                                                  Win32Methods.GetModuleHandle(curModule.ModuleName), 0);
        }
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposed)
        {
            if (disposing)
            {
                if (subject != null)
                {
                    subject.Dispose();
                }
            }

            disposed = true;
        }
    }

    class DelegateDisposable : IDisposable
    {
        readonly Action dispose;

        public DelegateDisposable(Action dispose)
        {
            this.dispose = dispose;
        }

        public void Dispose()
        {
            dispose();
        }
    }
}
```

There are a lot of things we did wrong, lets look at them one at a time.

# DelegateDisposable class
This is a simple win. Rx comes with a *heap* of really useful disposable types. We can delete our nested DelegateDisposable class and replace it's usage with `Disposable.Create()`. Quite an easy win.

# Implementing IObservable<InterceptKeyEventArgs>
When using Rx you should never have to implement `IObservable<T>` or `IObserver<T>`. If you do then you should be looking for a better way to do it.

Because we are implementing IObservable we have to provide a Subscribe method. Lets have a look at that.

    public IDisposable Subscribe(IObserver<InterceptKeyEventArgs> observer)
    {
        IDisposable dispose = subject.Subscribe(observer);
        subscriberCount++;
        if (subscriberCount == 1)
            hookId = SetHook(callback);
        return Disposable.Create(() =>
                          {
                              subscriberCount--;
                              if (subscriberCount == 0)
                                  Win32Methods.UnhookWindowsHookEx(hookId);
                              dispose.Dispose();
                          });
    }

The first side effect of this is that we have to do subscription management ourselves now! The good news is we can make a very simple change to improve this.

1. No longer inherit from IObservable<InterceptKeyEventArgs>
2. Rename subscribe to `GetKeyStream` with this signature:  
`IObservable<InterceptKeyEventArgs> GetKeyStream()`
3. Make it compile again.

To make it compile we need to create the observable that we are going to return. We will do that in the contructor of InterceptKeys so we always return the same observable to all callers (remember InterceptKeys is a singleton).

``` csharp
InterceptKeys()
{
    keyStream = Observable.Create<InterceptKeyEventArgs>(observer =>
    {
        hookId = SetHook(callback);
        var dispose = subject.Subscribe(observer)

        return Disposable.Create(() =>
        {
            Win32Methods.UnhookWindowsHookEx(hookId);
            dispose.Dispose();
        });
    })
    .Publish().RefCount();
}

public IObservable<InterceptKeyEventArgs> GetKeyStream()
{
    return keyStream;
}
```

Ok this is looking better! We no longer are doing subscription management ourself. Lets look at the new Rx features we have used.

### Observable.Create
If you ever need an observable, it is likely you want to use a static method off the `Observable` class. Observable.Create passes us an observable which we can call OnNext(), OnError() or OnComplete() on to yield new values to any subscribers.

You then return an IDisposable which is called when the subscription is disposed by the subscriber. 

### Publish()
The next new thing is the Publish extension method. It takes an IObservable and returns a IConnectableObservable. Connectable observables give the subscriber control over when the underlying source is actually connected to and disconnected from. As a quick example:

```csharp
var observable = Observable.Create<InterceptKeyEventArgs>(observer =>
    {
        Debug.WriteLine('Subscribed');
        hookId = SetHook(callback);
        var dispose = subject.Subscribe(observer)

        return Disposable.Create(() =>
        {
            Debug.WriteLine('Unsubscribed');
            Win32Methods.UnhookWindowsHookEx(hookId);
            dispose.Dispose();
        });
    });

// Without Publish
var subscription = observable.Subscribe(_ => { }); // Prints 'Subscribed'
subscription.Dispose(); // Prints 'Unsubscribed'

// With Publish
var published = observable.Publish();
var subscription = published.Subscribe(_ => { });
observable.Connect();   // Prints 'Subscribed'
```

### RefCount()
Most of the time you do not want to manage the connection/disconnection of a connected observable yourself. This is where `RefCount` extension method comes in. It simply counts the number of subscribers and when the observable has subscribers it connects the underlying Published observable. Once all subscribers have Disposed it will disconnect the Published observable.

Now Rx is fully managing our subscriptions and will make sure we only ever have a single low-level keyboard hook!

## Cleanup
Now that we have made this change, what else can be cleaned up?

``` csharp

decimal subscriberCount;

public void Dispose()
{
    Dispose(true);
    GC.SuppressFinalize(this);
}

protected virtual void Dispose(bool disposing)
{
    if (!disposed)
    {
        if (disposing)
        {
            if (subject != null)
            {
                subject.Dispose();
            }
        }

        disposed = true;
    }
}
```

We can delete all of this code now. 


# Using Subject<InterceptKeyEventArgs>
The next warning sign we should be looking for in our Rx code is the use of Subjects. A subject is both a IObservable<T> AND an IObserver<T>. In this case we have a subject stored in a field which the Low Level Keyboard hook publishes to, inside our keyStream observable we have subscribed to the subject using the observer that Observable.Create gave to us. 

To do this we have to move the low level keyboard hook into our Observable.Create so it can publish directly to the observer Rx has created us.

``` csharp
keyStream = Observable.Create<InterceptKeyEventArgs>(observer =>
{
    Debug.Write("Subscribed to keys");
    IntPtr hookId = IntPtr.Zero;
    // Need to hold onto this callback, otherwise it will get GC'd as it is an unmanged callback
    callback = (nCode, wParam, lParam) =>
    {
        if (nCode >= 0)
        {
            var eventArgs = CreateEventArgs(wParam, lParam);
            observer.OnNext(eventArgs);
            if (eventArgs.Handled)
                return (IntPtr)1;
        }

        return CallNextHookEx(hookId, nCode, wParam, lParam);
    };
    hookId = SetHook(callback);
    return Disposable.Create(() =>
    {
        Debug.Write("Unsubscribed from keys");
        UnhookWindowsHookEx(hookId);
        callback = null;
    });
})
.Publish().RefCount();
```

This is not the nicest code because of the API of `SetWindowsHookEx` but we have removed our Subject and the code is far easier to understand now.

# Summary
After those refactorings we have a far more maintainable class and we have removed a number of Rx anti-patterns. This is what the class looks like after those refactorings:

``` csharp
[PermissionSet(SecurityAction.LinkDemand, Name = "FullTrust")]
[PermissionSet(SecurityAction.InheritanceDemand, Name = "FullTrust")]
public class InterceptKeys : IInterceptKeys
{
    public static readonly InterceptKeys Current = new InterceptKeys();
    LowLevelKeyboardProc callback;
    readonly IObservable<InterceptKeyEventArgs> keyStream;

    private InterceptKeys()
    {
        keyStream = Observable.Create<InterceptKeyEventArgs>(observer =>
        {
            Debug.Write("Subscribed to keys");
            IntPtr hookId = IntPtr.Zero;
            // Need to hold onto this callback, otherwise it will get GC'd as it is an unmanged callback
            callback = (nCode, wParam, lParam) =>
            {
                if (nCode >= 0)
                {
                    var eventArgs = CreateEventArgs(wParam, lParam);
                    observer.OnNext(eventArgs);
                    if (eventArgs.Handled)
                        return (IntPtr)1;
                }

                return CallNextHookEx(hookId, nCode, wParam, lParam);
            };
            hookId = SetHook(callback);
            return Disposable.Create(() =>
            {
                Debug.Write("Unsubscribed from keys");
                UnhookWindowsHookEx(hookId);
                callback = null;
            });
        })
        .Publish().RefCount();
    }

    public IObservable<InterceptKeyEventArgs> GetKeyStream()
    {
        return keyStream;
    }

    InterceptKeyEventArgs CreateEventArgs(IntPtr wParam, IntPtr lParam)
    {
        /* snip, not really important, just calculating the variabless used below. */

        return new InterceptKeyEventArgs(key, keyDirection, alt, control, shift);
    }

    static IntPtr SetHook(LowLevelKeyboardProc proc)
    {
        //TODO: This requires FullTrust to use the Process class - is there any options for doing this in MediumTrust?
        //
        using (Process curProcess = Process.GetCurrentProcess())
        using (ProcessModule curModule = curProcess.MainModule)
        {
            return SetWindowsHookEx(WH_KEYBOARD_LL, proc, GetModuleHandle(curModule.ModuleName), 0);
        }
    }
}
```

Pretty good result in the end. We have reduced this file from 224 lines of code to 100 lines by not implmenting IObservable<T> ourselves, letting Rx manage our subscriptions (with Publish().RefCount()) and making sure we use types provided by Rx (Disposable.Create instead of our own type).

Next in this series we will be rewriting the class which takes key presses (ctrl + f, a, b, space etc) and turns it into the messages carnac shows on the screen. 