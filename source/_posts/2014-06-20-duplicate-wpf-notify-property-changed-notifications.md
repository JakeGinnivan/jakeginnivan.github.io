---
layout: post
published: false
title: Duplicate WPF Notify Property Changed Notifications
comments: true
---

The application I am currently working on has a complex WPF form for entering data, many of the fields cause other fields to have defaults calculated which can cause another field to recalculate.

The problem comes where some of the field defaults are server calls so I only want it to be done once at the end of the cascade. This post is all about reducing duplicate NotifyPropertyChanged events so it works the way you want, not the way it actually does now.

<!-- more -->

We are using Rx to cause the defaults to be calculated, it looks like this:

    this.ObservePropertiesChanged(vm => vm.Property1, vm => vm.Property2)
        .Subscribe(_ => CalculateProperty3());
    
    this.ObservePropertyChanged(vm => vm.Property3).Subscribe(_ => CalculateProperty4());
    
    this.ObservePropertiesChanged(vm => vm.Property3, vm => vm.Property4)
        .Subscribe(_ => SetProperty5FromServer());
    
    this.ObservePropertiesChanged(
        vm => vm.Property1,
        vm => vm.Property2,
        vm => vm.Property3,
        vm => vm.Property4,
        vm => vm.Property5)
        .Subscribe(_ => CalculateSummary());
        
This looks pretty simple, but lets think about what will happen when we set Property1.

Property1 = "Foo" **==>**  
Triggers `CalculateProperty3()` **==>**  
Triggers `CalculateProperty4()` and `SetProperty5FromServer()` **==>**  
`CalculateProperty4()` will cause `SetProperty5FromServer()` to be triggered for a second time **==>**  
CalculcateSummary will be called 3 times, and when Property5 comes back from the server it will be calculated again

That is a bunch of things happening which I do not actually want to.

What I actually want to happen is:
 - Allow me to react to property changes before they happen and calculate the cascades and set the properties before any INotifyPropertyChanged events are raised
 - When observing multiple properties only OnNext when a property is different. So if I am listening to Property1 and Property2 then I set both and raise the property changed event for both properties I want OnNext to fire *once*

## A solution
The first step is to create a new event in your ViewModelBase, I called it `PropertyBeingUpdated`

    public event Action<string> PropertyBeingUpdated = notification => { };

Now lets modify our OnPropertyChanged() method to raise this event first:

    [NotifyPropertyChangedInvocator]
    public void OnPropertyChanged(string propertyName)
    {
        PropertyBeingUpdated(propertyName);
    }

Next, we need to track all the properties which are trying to raise property changed notifications. 

    private readonly List<string> raising = new List<string>();
    
    [NotifyPropertyChangedInvocator]
    public void OnPropertyChanged(string propertyName)
    {
        raising.Add(propertyName);

        PropertyBeingUpdated(propertyName);
    }

Great, now when anyone listening to PropertyBeingUpdated sets a property, it will call back into this method and the raising collection will have both properties in it.

Now to raise the PropertyChanged event.

    [NotifyPropertyChangedInvocator]
    public void OnPropertyChanged(string propertyName)
    {
        var isFirst = raising.Count == 0;
        raising.Add(propertyName);

        PropertyBeingUpdated(propertyName);

        if (isFirst)
        {
            var raisingList = raising.Distinct().ToArray();
            raising.Clear();
            foreach (var toRaise in raisingList)
            {
                PropertyChanged(this, new PropertyChangedEventArgs(toRaise));
            }
        }
    }

Now lets update our original subscriptions so use this new event with a new extension method `ObservePropertiesBeingChanged`

    this.ObservePropertiesBeingChanged(vm => vm.Property1, vm => vm.Property2)
        .Subscribe(_ => CalculateProperty3());
    
    this.ObservePropertiesBeingChanged(vm => vm.Property3).Subscribe(_ => CalculateProperty4());
    
    // Any server calls should still listen to PropertyChanged
    this.ObservePropertiesChanged(vm => vm.Property3, vm => vm.Property4)
        .Subscribe(_ => SetProperty5FromServer());
    
    this.ObservePropertiesBeingChanged(
        vm => vm.Property1,
        vm => vm.Property2,
        vm => vm.Property3,
        vm => vm.Property4,
        vm => vm.Property5)
        .Subscribe(_ => CalculateSummary());

## Distinct ObservePropertiesBeingChanged
The next issue is the fact that if Property2 and Property3 are both set, then the changed notifications are raised I will calculate the summary twice (because there were two events). I actually only want it once. So we can make ObservePropertiesBeingChanged a bit smarter to only raise when any of the values being observed has changed since the last time.

    public static IObservable<Unit> ObservePropertiesBeingChanged<TSource>(
        this TSource source,
        params Expression<Func<TSource, object>>[] propertyExpressions) where TSource : IPropertyBeingUpdated
    {
        var selectors = propertyExpressions.Select(CompiledExpressionHelper<TSource, object>.GetFunc).ToArray();
        return propertyExpressions
            .Select(expr => ObservePropertyBeingChanged(source, expr).Select(_ => Unit.Default))
            .Merge()
            .Select(_ => selectors.Select(s => s(source)).ToArray())
            .DistinctUntilChanged(new ArrayEqualityComparer<object>())
            .Select(_ => Unit.Default);
    }
    
The smarts are in these two lines:

    .Select(_ => selectors.Select(s => s(source)).ToArray())
    .DistinctUntilChanged(new ArrayEqualityComparer<object>())

Here we select all the values of those properties into an array, we then use Rx's DistinctUntilChanged operator with a custom comparer which compares all the values in the array. 

The full source for these extension methods are at [https://gist.github.com/JakeGinnivan/a8dd02203cb871a9c16a](https://gist.github.com/JakeGinnivan/a8dd02203cb871a9c16a)

Hopefully this is useful to you.

