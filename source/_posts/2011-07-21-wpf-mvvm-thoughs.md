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

#My approach

I find that once you progress past a basic application with simple interactions that often you need views to interact with each other, this could be returning data from a modal dialogue or anything.

One thing we are often told is we should develop against interfaces, not implementations. I think this applies to views as well, so lets create an IView interface.

    public interface IView
    {
    }

Now I want to be able to tell the calling viewmodel if I was cancelled, or possibly return data, so we will create three more interfaces and a few classes:

    public interface IDialogueView : IView
    {
        void DialogueDisplayed(IDialogueWindow window);
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
            
        }
    }