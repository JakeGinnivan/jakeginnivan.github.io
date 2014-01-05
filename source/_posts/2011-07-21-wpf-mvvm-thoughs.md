---
layout: post
title: MVVM, Design Time vs IoC usage
metaTitle: MVVM, Design Time vs IoC usage
description: After a discussion via twitter I thought I would post my thoughts around how I would wire up views/viewmodels when not using a specific framework
revised: 2011-07-22
date: 2011-07-21
categories: [c#,wpf]
migrated: true
comments: true
sharing: true
footer: true
permalink: /wpf-mvvm-thoughs/
summary: | 
  After a discussion via twitter I thought I would post my thoughts around how I would wire up views/viewmodels when not using a specific framework

---
# The problems

Lets start with the problems we want to solve with our solution

 - Blend Design time support
 - Resharper binding validation support
 - No using ServiceLocator, DI only
 - ViewModels should be testable

Frameworks like Magellan solve these sort of issues in other ways, but if you don't want to take on a framework then maybe an approach like this may help you get started.
<!-- more -->
#My approach

I find that once you progress past a basic application with simple interactions that often you need views to interact with each other, this could be returning data from a modal dialogue or anything.

One thing we are often told is we should develop against interfaces, not implementations. I think this applies to views as well, so lets create an IView interface.

    public interface IView
    {
    }

Now I want to be able to tell the calling viewmodel if I was cancelled, or possibly return data, so we will create three more interfaces and a few classes:

    public interface IDialogueView : IView
    {
        void DialogueDisplayed();
        void DialogueClosed();
    }

    public interface IDialogueView<TResult> : IDialogueView
    {
        event EventHandler<DialogueResultEventArgs<TResult>> Finished;
    }

    public interface IDialogueViewWithoutResult : IDialogueView
    {
        event EventHandler<DialogueResultEventArgs> Finished;
    }

    public class DialogueResultEventArgs : EventArgs
    {
        public DialogueResultEventArgs(bool cancelled)
        {
            Cancelled = cancelled;
        }

        public bool Cancelled { get; private set; }

        public static DialogueResultEventArgs EmptyResult = new DialogueResultEventArgs(false);
        public static DialogueResultEventArgs CancelledResult = new DialogueResultEventArgs(true);
    }

    public class DialogueResultEventArgs<T> : EventArgs
    {
        public DialogueResultEventArgs(Exception error)
        {
            Result = new DialogueResult<T>(error);
        }

        public DialogueResultEventArgs(T result)
        {
            Result = new DialogueResult<T>(result);
        }

        public static DialogueResultEventArgs<T> Cancelled()
        {
            return new DialogueResultEventArgs<T> { Result = DialogueResult<T>.CancelledResult() };
        }

        private DialogueResultEventArgs() { }

        public DialogueResult<T> Result { get; private set; }
    }



Then we need some way for our ViewModels to interact with other Views using these interfaces, enter IUIService

    public interface IUIService
    {
        /// <summary>
        /// Shows a view in dialogue mode. The view will return a result. This call is blocking.
        /// </summary>
        /// <param name="dialogueView">The dialogue view.</param>
        DialogueResult<TResult> ShowDialogue<TResult>(IDialogueView<TResult> dialogueView);

        /// <summary>
        /// Shows a view in dialogue mode. No result will be returend. This call is blocking.
        /// </summary>
        /// <param name="dialogueView">The dialogue view.</param>
        DialogueResult ShowDialogue(IDialogueViewWithoutResult dialogueView);
    }

## Creating a new view
One of our requirements is that we want to support blend, this is pretty easy but I think that the view shells should always be created by the developers. If you have designers, create the empty view for them, wire it up, then hand over to them for them to do their magic. As much as Microsoft has made the developer/designer story much better, you still have to weigh up the pros and cons of trying to get the designers to do too much. Either they learn a basic bit of coding to get a new view setup, or they rely on the devs to create and wire up the empty view.

Start off and create a new 'WPF Window'. Then we define an interface for it in the code behind.

    public partial class MyView : IMyView
    {
        public MyView(MyViewModel viewModel)
        {
            _viewModel = viewModel;
            DataContext = _viewModel;
            InitializeComponent();
        }

        public void DialogueDisplayed()
        {
            _viewModel.Initialise();
        }

        public void DialogueClosed()
        {
        }

        public event EventHandler<DialogueResultEventArgs<SomeData>> Finished
        {
            add { _viewModel.Finished += value; }
            remove { _viewModel.Finished -= value; }
        }
    }

    public interface IMyView : IDialogueView<SomeData>
    { }

Doing this requires some basic coding, but allows full IoC support, and allows the viewmodel to raise the finished event, which will cause the UIService to close the window and return to the caller. It also gives a nice place to hook into things. 
In my current projects, we actually define all windows as UserControls, and our UI server creates the window with custom chrome and gives us full control over everything. It is working very well.

Then lets look at the Xaml.

    <Window x:Class="WpfApplication3.MainWindow"
            xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
            xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
            Title="MainWindow" Height="350" Width="525"
            xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
            xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
            mc:Ignorable="d"
            d:DataContext="{d:DesignInstance MainViewModel, IsDesignTimeCreatable=True}">
        <Grid>
        
        </Grid>
    </Window>

In my current project we also have an attached behaviour which allows us to specify a provider to set the datacontext. We leave the d:DataContext in there so we get R# support for our bindings =)

Next chance I get, I was thinking that doing something like this in the viewmodel might work quite well.

# IoC - Putting it together

In our appliation startup, we need to register out IoC container, I am using Autofac. This is my registration

    private void ApplicationStartup(object sender, StartupEventArgs e)
    {
        var containerBuilder = new ContainerBuilder();
        containerBuilder.RegisterType<UIService>().As<IUIService>();
        containerBuilder.RegisterType<SomeService>().As<ISomeService>();
        containerBuilder
            .RegisterAssemblyTypes(typeof(App).Assembly)
            .AssignableTo<IView>()
            .AsImplementedInterfaces();
        containerBuilder
            .RegisterAssemblyTypes(typeof(App).Assembly)
            .AssignableTo<ViewModelBase>()
            .AsSelf();
        var container = containerBuilder.Build();
        container.Resolve<IUIService>().ShowDialogue(container.Resolve<IMainView>());
    }

I register my UIService, another random service, then use the assembly scanning features to register viewmodels and views. I then resolve my UIService, and show my IMainViewModel!


#Design Time and Runtime ViewModel wireup
`Note:` I have not used this technique in a production project, but think it would be cool to try out and think it would be better than the setup I am using now.

We add default constructor to our viewmodel:

    public MainWindow() : this(new MainViewModel())
    {
    }

We then setup our viewmodel, which will look like:

    public class MainViewModel : ViewModelBase
    {
        public MainViewModel()
        {
            this.PopulateDesignTimeData().With<MainViewModelData>();
        }

        public MainViewModel(ISomeService someService)
        {
            //To show some DI at runtime
            SomeProperty = someService.GetAValue();
        }

        public string SomeProperty { get; set; }
        public event EventHandler<DialogueResultEventArgs> Finished;

        protected void OnFinished(DialogueResultEventArgs e)
        {
            var handler = Finished;
            if (handler != null) handler(this, e);
        }
    }

Notice in our default contructor we call `this.PopulateDesignTimeData().With<MainViewModelData>()`? Here is the MainViewModelData class:

    public class MainViewModelData : IDesignTimeViewModelPopulator<MainViewModel>
    {
        public void Populate(MainViewModel viewModel)
        {
            viewModel.SomeProperty = "blah";
        }
    }

Because of the way our IoC containers work, they will always choose the most specific constructor they can. If it cannot find the dependencies it will default back to the default constructor, which will throw a Debug.Assert because it is not in design time mode.

# Source

[Download example project](/get/downloads/WpfApplication3.zip)