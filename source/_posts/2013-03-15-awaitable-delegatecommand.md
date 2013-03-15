---
layout: post
title: Awaitable DelegateCommand
metaTitle: Awaitable DelegateCommand
description: Use MVVM, DelegateCommand, testing and c# 5 async features. You want to see this
revised: 2013-03-15
date: 2013-03-15
categories: [mvvm,async,delegatecommand]
migrated: true
comments: true
sharing: true
footer: true
permalink: /awaitable-delegatecommand/
summary: | 
  Use MVVM, DelegateCommand, testing and c# 5 async features. You want to see this

---
Most of you would be aware of DelegateCommand, it allows you to turn a lambda (or two if you want to specify canexecute) into a WPF ICommand.

This is pretty easy to unit test as well, you can simply go `viewModel.MyCommand.Execute(null)`

But this falls down when you start using async, because the lambda will end up being an **async void** method. This means your test can no longer observe what happens in that command. If the implementation throws, your unit test will not know about it and then your test process will crash (if using .net 4.0).

Read [Lucians blog post](http://blogs.msdn.com/b/lucian/archive/2013/02/18/talk-the-new-async-design-patterns.aspx) for more information about why async void is bad.

So we introduced a new interface, called `IAsyncCommand` and it looks like this

    public interface IAsyncCommand : IAsyncCommand<object>
    {
    }

    public interface IAsyncCommand<in T> : IRaiseCanExecuteChanged
    {
        Task ExecuteAsync(T obj);
        bool CanExecute(object obj);
        ICommand Command { get; }
    }

If you were wondering what IRaiseCanExecuteChanged is, it is a really simply interface which means we don't have to cast ICommands. See this [Gist](https://gist.github.com/JakeGinnivan/5166866) for the code for that.

You might notice that there is no `void Execute(object obj)` method on the command, this is to make it impossible to call the async void overload in your unit tests, unless you get the underlying ICommand from the Command property.

# So how does WPF execute the command?
WPF actually doesnt care about what is exposed via the API, as long as the class implements ICommand, we are good. Introducing AwaitableDelegateCommand:

    public class AwaitableDelegateCommand : AwaitableDelegateCommand<object>, IAsyncCommand
    {
        public AwaitableDelegateCommand(Func<Task> executeMethod) 
            : base(o=>executeMethod())
        {
        }

        public AwaitableDelegateCommand(Func<Task> executeMethod, Func<bool> canExecuteMethod) 
            : base(o=>executeMethod(), o=>canExecuteMethod())
        {
        }
    }

    public class AwaitableDelegateCommand<T> : IAsyncCommand<T>, ICommand
    {
        private readonly Func<T, Task> _executeMethod;
        private readonly DelegateCommand<T> _underlyingCommand;
        private bool _isExecuting;

        public AwaitableDelegateCommand(Func<T, Task> executeMethod)
            : this(executeMethod, _ => true)
        {
        }

        public AwaitableDelegateCommand(Func<T, Task> executeMethod, Func<T, bool> canExecuteMethod)
        {
            _executeMethod = executeMethod;
            _underlyingCommand = new DelegateCommand<T>(x => { }, canExecuteMethod);
        }

        public async Task ExecuteAsync(T obj)
        {
            try
            {
                _isExecuting = true;
                RaiseCanExecuteChanged();
                await _executeMethod(obj);
            }
            finally
            {
                _isExecuting = false;
                RaiseCanExecuteChanged();
            }
        }

        public ICommand Command { get { return this; } }

        public bool CanExecute(object parameter)
        {
            return !_isExecuting && _underlyingCommand.CanExecute((T)parameter);
        }

        public event EventHandler CanExecuteChanged
        {
            add { _underlyingCommand.CanExecuteChanged += value; }
            remove { _underlyingCommand.CanExecuteChanged -= value; }
        }

        public async void Execute(object parameter)
        {
            await ExecuteAsync((T)parameter);
        }

        public void RaiseCanExecuteChanged()
        {
            _underlyingCommand.RaiseCanExecuteChanged();
        }
    }

Notice that the AwaitableDelegateCommand implements ICommand, which makes WPF happy, but our viewmodels expose IAsyncCommand, which makes our tests happy. All in all, it works pretty well for me!

If you need a `DelegateCommand` implementation, check out [this gist](https://gist.github.com/JakeGinnivan/5166898)