---
layout: post
title: Windows Phone MVC Update
metaTitle: Windows Phone MVC Update
description: I have just pushed a MASSIVE update! So many enhancements, bug fixes, and maturity is in this release. 
revised: 2011-08-22
date: 2011-08-21
categories: [wp7,mvc,open-source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /windows-phone-mvc-update/
summary: | 
  I have just pushed a MASSIVE update! So many enhancements, bug fixes, and maturity is in this release. 

---
# Windows Phone MVC?
So to get started, let me explain what it is.  
It is a MVC + MVVM Hybrid framework, I think MVVM falls down in a few area's, and by teaming up with the MVC pattern we can achieve great testability, performance, navigation, lifecycle etc.. 

The aim of Windows Phone MVC is to make windows phone development quicker, easier, more enjoyable and most of all help you build a nice performant app which gives an awesome user experience.

This is going to be quite a long post going through many features of Windows Phone MVC, I hope it gets across what I am trying to do with the framework, and how it can help you out!  
I am still a little way from v1, so there will be API changes, but I am more than happy to field some questions about how to use it.

Many of the improvements on this release are due to [http://transhub.wordpress.com/](http://transhub.wordpress.com/) using Windows Phone MVC and I have been working close with the transub team to make sure the framework helped them deliver a killer app!

## MVC
First off it is important to understand how a MVC framework works.

It all starts off with a Controller, and an Action.  

    public class HomeController : Controller
    {
        public ActionResult Main()
        {
            return Page(new MainViewModel());
        }
    }

An **action** is simply a method on a class which inherits from Controller. The Controller returns an ActionResult, there are helper methods in the controller base class which let you do this with ease.

**Note:** is actions are invoked on a `background thread` and you should do work synchronously, we have made this easy with:

    public ActionResult GetWebData()
    {
        var request = WebRequest.CreateHttp(url);
        var response = (HttpWebResponse)Execute.AsyncPatternWithResult(request.BeginGetResponse, request.EndGetResponse);
        return Result(response.GetResponseString());
    }

Will talk a bit more about threading later..

### ActionResults
We have a few different action results, they include:

 - PageResult (Navigates to a new page, the parameter you pass becomes the datacontext for that page)
 - DialogResult (Shows a dialog over the top of your current page)
 - NothingResult (Allows you to finish a controller action without doing anything. i.e if (TrialExpired) `return Nothing();`
 - BackResult (navigates back)
 - BackToResult (navigates back to a particular view. i.e to go back to home, no matter how many between `return BackTo<HomeController>("Main");`)
 - DataResult (used to return data to the previous view, more info below!)

Using these, you can do most things you need to pretty easily.

### Invoking actions
Generally you will invoke ALL actions from the viewmodel, currently all the API's are named around navigation, this may change in the future. What do you think?

    //MainViewModel ctor
    public MainViewModel()
    {
        SearchCommand = Controller<SearchController>.NavigationCommand(c=>c.SearchLocations(SearchCriteria));
    }

    public SearchCriteria SearchCriteria { get { return searchCriteria; } }
    public ICommand SearchCommand { get; private set; }

## Performance
One of the first things that people say to me when I say I am building *another* framework for windows phone is, 'I don't use frameworks, they are too slow'.  
Something to keep in mind is that there are two types of performance, actual performance and perceived performance. The latter is actually the more important of the two, because it is how the user perceives how fast the application is.
I remember reading in a book the story about how back in the day Microsoft got told that their c++ compiler was slow vs the borland one, even though it WAS faster, so they then made it output everything that it was happening onto the console and the users congratulated them on the improvement. It was about 5% slower because of the output.  

Windows Phone MVC tries to achieve both by default, I am trying to optimise the framework as much as I can, the overhead in startup time & memory usages is 15-20% at the moment based on some initial testing, I hope to drop this over time and put some neat perf tricks in.

![Progress Screen](/assets/posts/2011-08-21-windows-phone-mvc-update/ProgressBar.png)

There is a few things to note here, Windows Phone MVC has a rich navigation system, so you can do work, load data, THEN navigate. You can also invoke an action which simply returns you data, or navigates to a page, then returns you the result from that page!

 1. Performance Progress Bar included out of the box (original, if you add transitions extension which needs Silverlight Toolkit, we switch to using the new one from Silverlight Toolkit).
 2. As soon as you invoke any action the Progress bar will be shown, and the screen grayed. You can also easily show text by just going `LoadingMessage("Search in progress..");` from your controller action.
 3. The app bar buttons are all disabled (we also give you full Commanding support in the app bar! more info on that later).

I still have some tuning and plan on rounding up some WP7 performance experts at TechEd to give me more tips about how I can improve the performance of this, but it feels and looks nice at the moment. 

## MVVM/Commanding Support
Out of the box there are some really nice helpers for you.

### Buttons

    <Button commands:Click.Command="{Binding AboutPageCommand}"
            Content="About" />
    <Button commands:Navigate.To="Home.ViewItem"
            DataContext="{Binding }" <!--Action can be ViewItem(ItemViewModel item)-->
            Content="View Item" />

The second can optionally pass the datacontext to the controller action, really useful when you have a button in a list of items.

### App Bar
As you probably know, the Application Bar is not a DependencyObject, so we can't use any of the nice silverlight goodness we are used to. I have taken the following approach:

    <phone:PhoneApplicationPage.ApplicationBar>
        <shell:ApplicationBar IsVisible="True"
                              IsMenuEnabled="True">
            <shell:ApplicationBarIconButton IconUri="/Icons/dark/appbar.add.rest.png"
                                            Text="new[AddCommand]" />
            <shell:ApplicationBar.MenuItems>
                <shell:ApplicationBarMenuItem Text="settings[ShowSettingsPageCommand]" />
                <shell:ApplicationBarMenuItem Text="about[ShowAboutPageCommand]" />
            </shell:ApplicationBar.MenuItems>
        </shell:ApplicationBar>
    </phone:PhoneApplicationPage.ApplicationBar>

Simply put the command name in square brackets in your appbar text, and they will be wired up to your viewmodel. Including listening to CanExecuteChanged to enable and disable appbar buttons!

### ListBoxes
Another handy addition is if you have a list of items, and you want to view details when any one of them is selected.

    <ListBox ItemsSource="{Binding Items}"
                Commands:Navigate.OnItemSelected="AnotherController.ItemAction"
                DisplayMemberPath="Title" />

### On the ViewModel
How often do you write something like `new DelegateCommand(()=>DoStuff())` in the case of MVC it would be `new DelegateCommand(()=>Controller<HomeController>().DoStuff())`. That is kinda ugly, so I have provided a nicer API for defining Commands:

    ShowGreetingPageCommand = Controller<HomeController>().NavigationCommand(c => c.SayHelloTo(Name));
    //or if you want to fetch some data/refresh the page: 
    RefreshCommand = Controller<AnotherController>().NavigationCommandWithResult<ObservableCollection<SearchResult>>(c => c.PerformSearch(Parameter), HandleCallback);

## Navigation
I mentioned earlier that Windows Phone MVC has a really powerful navigation system. It actually does not use the inbuild NavigationService, it navigates once at the start of the all to your `Shell.xaml`, which hosts your app, the framework then takes care of switching out the content, and maintaining it's own journal. This has a number of advantages.

Be aware that Navigation with Windows Phone MVC is equivilent to Tombstoning that view, when you navigate the controller action will ALWAYS be executed. The advantage of this is that your views/viewmodels can be garbage collected as soon as you navigate away from that view.

### Type safe navigation WITH arguments
Take this navigation:

    Controller<HomeController>().NavigateTo(c => c.MyAction(ComplexProperty)); 

This is all type safe, and I am passing any class as an argument to my action, I don't need to know the view name, and I get full intellisense on the arguments.

**NOTE:** Due to WP7 limitations around reflection, you cannot pass local variables (declare in the same method), method arguments, or protected/private fields. All of these things require reflection on non public types, which is not supported.   
Never fear!

If you try and do this:

    private void NavigateWithPrivate()
    {
        var local = new SomeClass();
        Controller<AnotherController>().NavigateTo(c=>c.PerformSearch(local));
    }

You will get a nice helpful error message:

![Useful exception message](/assets/posts/2011-08-21-windows-phone-mvc-update/NavNotSupported.png)

You can literally copy and paste the suggested syntax (check it first, and let me know if I get any cases wrong) over your code, then it will start working! 

### Back stack manipulation (without Mango :P)
At any time in a viewmodel or a controller you can write:

    Navigator.RemoveBackEntry();

That is pretty cool, and works without mango. But to go one better than what you get in Mango:

    Navigator.NavigateBackTo<HomeController>("Main");
    // or in controller:
    return BackTo<HomeController>("Main");

This will unwind the backstack until you get to the matching controller and action! If it can't find it, it will let you know.

### Returning data to previous page
A great example of this is the DateTimePicker in the toolkit, when you click on it, it actually hijacks the navigation service, navigates to a page inside the toolkit, then when you click on one of the appbar buttons it navigates back, then updates the control. Sound useful? It really is..

#### Without a navigation
This performs a search in a controller action, then returns the data without navigating.

    private void PerformSearch()
    {
        Controller<SearchController>().NavigateToWithResult<JourneyPlanningResponse>(c => c.PerformSearch(SearchCriteria), SearchComplete);
    }

    private void SearchComplete(JourneyPlanningResponse response)
    {
         ....
    }

#### With navigation
What about the datepicker example:

    GetValueCommand = Controller<DebugController>().NavigationCommandWithResult<string>(c => c.PageWithResult(), HandleCallback);

Then on our viewmodel looks like this:

    public class PageWithResultViewModel : ViewModelBase
    {
        private string valueToReturn;

        public PageWithResultViewModel()
        {
            OkCommand = new DelegateCommand(() => Navigator.NavigationResult(ValueToReturn));
            CancelCommand = new DelegateCommand(() => Navigator.NavigateBack());
        }

        public string ValueToReturn
        {
            get { return valueToReturn; }
            set
            {
                valueToReturn = value;
                OnPropertyChanged(()=>ValueToReturn);
            }
        }

        public ICommand OkCommand { get; private set; }
        public ICommand CancelCommand { get; private set; }
    }

I recon that is pretty cool! And super easy to use.... 

## Threading
Mentioned earlier, actions are invoked on a background thread, and the Controller's lifetime scope (if using autofac) is disposed as soon as the action is invoked, so you shouldn't `return` from the  action before you have fully setup the new view's viewmodel and you shouldn't be waiting for any callbacks. The way we have solved this is with the `Execute` class.

    var response = (HttpWebResponse)Execute.AsyncPatternWithResult(svc.BeginGetResponse, someArg, request.EndGetResponse);
   //or if the call doesnt return anything
   Execute.AsyncPattern(svc.BeginDoSomething, someArg, svc.EndDoSomething);

This will turn the async pattern Begin/End pair into a synchronous call. You should not use this class on the UI Thread obviously...

Talking about the UI Thread, what happens if you want to do stuff on the UI Thread?

    Execute.OnUIThread(()=>{/*dostuff*/});
    Execute.OnUIThreadSync(()=>{/*do stuff synchronously*/});

These two methods invoke an action(or func<T> with the synchronous option) on the UI Thread asynchronously and synchronously respectively.

## WP7 Lifecycle
Another very important part of developing for WP7 is the lifecycle management. We do a fair bit here to help you out.

If you want a property to be saved between navigations, and tomb-stoning simply mark it as transient like follows:

    public RegisterViewModel()
    {
        Transient(() => Name);
        Transient(() => Username);
        Transient(() => Password);
    }

Because we recreate the view each time you navigate (I may possibly introduce caching of the view so it doesn't have to be recreated, but memory pressure is tight pre-mango so will investigate in the future), Panoramas will not restore to the same panel that was selected. This is easily fixed with an attached property

    <toolkit:Panorama Title="my application"
                      Phone:Restore.PanoramaPosition="True">

Let me know if there any other things that you would like restored.

### OnActivate/OnDeactivate
Windows Phone MVC changes the behaviour slightly from the default experience. When a view is navigated away from, OnDeactivate will be called, and whenever it is shown OnActivate will be executed. Simply override one/both of these methods to do anything you need to for tombstoning.

### PageState
Earlier I mentioned that Windows Phone MVC has it's own NavigationService, and Journal. This means I cannot use the included PageState from the framework as it is tightly coupled with the phones NavigationService. So I have rolled my own.

    public interface IPageTransientStore
    {
        void Save<T>(string key, T obj) where T : class;
        T Load<T>(string key) where T : class;
        bool Contains(string key);
        void Remove(string key);
    }

All serialisation in Windows Phone MVC is done using Mike Talbot's [Silverlight Serializer](http://whydoidoit.com/silverlight-serializer/). This means I can serialise complex types, without specifying known types very fast.

### Obscured
If you are interested in being notified when your app is obscured, simple make your ViewModel inherit from `IObscuredAware`, like so:

    public class ViewResultTestViewModel : ViewModelBase, IObscuredAware
    {
        public void Unobscured(object sender, EventArgs e)
        {
        }

        public void Obscured(object sender, ObscuredEventArgs e)
        {
        }
    }

## Deep Linking
We also support Mango features! If you want to create a deep link into your Windows Phone MVC application, that is really easy too.. It is also type safe =)

    private void CreateDeepLink()
    {
        var deepLinkUri = Controller<HomeController>().UriFor(c => c.DeepLinkPage, new Dictionary<string, string>());
        ShellTile.Create(new Uri(deepLinkUri, UriKind.Relative), new StandardTileData
                                                    {
                                                        BackgroundImage = new Uri("/ApplicationIcon.png", UriKind.Relative),
                                                        Title = "MVC Deep Link"
                                                    });
    }

Deep links can only take a collection of KeyValuePair<string, string> for the parameters. 

## Transitions
Everyone loves a nice sexy app that has transitions. Jeff and the guys behind the Silverlight have made it pretty easy to get nice transitions working in your app. You will be happy to know it is just as easy for Windows Phone MVC. Simple run in your NuGet console `Install-Package WindowsPhoneMVC.Extensions.Transitions`. This relies on the Silverlight toolkit and will drop a `To_Enable_Transitions.txt` file into your project with the code required to wire it up (super easy).
The usage is exactly the same after the setup.

## IoC Container Support
By default, Windows Phone MVC has a DefaultControllerFactory, this isn't very interesting. So you can just run this in your NuGet console `Install-Package WindowsPhoneMVC.Extensions.AutofacIntegration` and you will get two files and a reference added to your project, the first is `To_Enable_Autofac.txt`, it will have code in it which you can copy/paste into App.xaml to enable the integration. The next is `ApplicationModule.cs` which is where you can register everything.

### Performance
Once again, as soon as you mention an IoC container on the phone, people freak out. The performance penalty is next to nothing if you do a bit of work in your ApplicationModule class.

Use Register(()=>) rather than RegisterType. Out of the box (so it works by default) you have this in your ApplicationModule.

    builder
        .RegisterAssemblyTypes(Assembly.GetExecutingAssembly())
        .AssignableTo<Controller>()
        .AsSelf();

This has the cost of reflecting your assembly when starting your app, then the reflection cost of construcing the object. A more performant alternative is:

    builder
        .Register(c=>new MainController(c.Resolve<ISomeService>()))
        .AsSelf();
    builder
        .Register(c=>new SomeService())
        .As<ISomeService>()
        .SingleInstance(); //Lightweight services can be left as SingleInstance

Sure this has more maintenance costs, but the performance different is substantial. On my phone the a object graph of ~90 object constructions took about 30% of the time vs letting the container do the hard work.

## Debugging 
To make diagnosing issues, and to give warnings and suggestions, simply handle the Trace event on the NavigationApplication and write the message out to Debug.Write. Internally no logging will be executed if the debugger is not attached.

    <Application.ApplicationLifetimeObjects>
        <AutofacIntegration:AutofacNavigationApplication Activated="AutofacNavigationApplicationActivated"
                                                         Deactivated="AutofacNavigationApplicationDeactivated"
                                                         Closing="NavigationApplication_OnClosing"
                                                         Trace="AutofacNavigationApplication_Trace"/>
    </Application.ApplicationLifetimeObjects>

In the HelloWorld application this produces an output like this:

    Navigator: Memory usage before navigation: 8.359375
    Navigator: Memory usage after navigation: 9.42578125
    Navigator: Memory usage before navigation: 9.46484375
    Navigator: Memory usage after navigation: 15.55859375
    Navigator: Memory usage before navigation: 13.84765625
    Navigator: Passing ViewModels as parameters may cause memory pressure as they are not disposed! Consider using base type of NotifyPropertyChanged if the parameter is actually an entity, not a ViewModel.
    Navigator: Memory usage after navigation: 19.08203125
    Navigator: Memory usage before navigation: 11.46875
    Navigator: Memory usage after navigation: 12.62890625
    Navigator: Memory usage before navigation: 11.828125
    Navigator: Memory usage after navigation: 16.98828125
    TimerScope: Beginning Parsing Navigation Expression
    Navigator: Memory usage before navigation: 15.3203125
    TimerScope: Ending Parsing Navigation Expression. Took 13ms
    Navigator: Memory usage after navigation: 19.84765625

Logging is pretty light at the moment, as I try and identify performance bottlenecks and other things I will likely introduce logging levels and lots more of it. I also hope to add helpful warnings for best practices as I have done with the note about ViewModels.

The downside of such a rich navigation model, is that you can pass very large things around, and parameters are always kept in memory, and viewmodels often have a lot more data than actually needs to be kept in memory.


# Conclusion
I hope this is a useful post as a introduction to Windows Phone MVC, and shows off many of the features which make it really easy to write WP7 applications.

Please leave feedback, ping me on twitter (@JakeGinnivan), or email me at jake@ginnivan.net with suggestions/issues. I want to power towards v1.0 when I will lock down API's and consider the framework stable. The framework is reasonably stable now, and is being used in a decent size app. But there will likely be situations I haven't thought of, or haven't tested. 

# Go get it!
Simply add `WindowsPhoneMVC` to your project via NuGet to get started, there will be a readme added to your project and a sample controller/ViewModels etc. If you don't want the files to get you started, choose the libs project