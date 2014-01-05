---
layout: post
title: Retry.For
metaTitle: Retry.For
description: I have created a simple class to help with UI Automation, making it really easy to retry a action for a certain time
revised: 2011-03-21
date: 2011-03-20
categories: [UI Automation]
migrated: true
comments: true
sharing: true
footer: true
permalink: /retry-for-class/
summary: | 
  

---
With UI automation, you are constantly dealing with race conditions, especially in a good WPF app which lazy loads most things, and while we have put a lot of good helpers in place (WaitWhileBusy etc), there are still times where a button does not enable quick enough, or a windows is opening and if you do not have automatic retry enabled you will have quite fragile UI automation tests.

This is where my Retry static class comes in handy.
<!-- more -->
##Usage
    var element = Retry.ForDefault(
                      () => screen.WhiteWindow.Get(searchCriteria),
                      b => (bool)b.AutomationElement.GetCurrentPropertyValue(AutomationElement.IsOffscreenProperty, false));

or for retrying if a locked log file is encountered (app still shutting down, or in middle of a write)

    Retry.For(() => File.Exists(@"C:\Logs\Terminal.log"), 5)


It will automatically retry if a value type is default, i.e if you call

    Retry.For<int>(()=>default(int), 5); //Will retry for 5 seconds, then return default(int)

And some more of the default conditions that cause retry

 - Exception thrown 
 - Value type == default(type) 
 - Reference type == null
 - bool == false

You can also override the default conditions with a predicate.

##Code

public static class Retry
    {
        public const int WindowWaitDefault = 30;
        public const int ElementWaitDefault = 10;

        public static Window ForDefault(Func<Window> getMethod)
        {
            return For(getMethod, WindowWaitDefault);
        }

        public static T ForDefault<T>(Func<T> getMethod)
        {
            return For(getMethod, ElementWaitDefault);
        }

        public static Window ForDefault(Func<Window> getMethod, Predicate<Window> shouldRetry)
        {
            return For(getMethod, shouldRetry, WindowWaitDefault);
        }

        public static T ForDefault<T>(Func<T> getMethod, Predicate<T> shouldRetry)
        {
            return For(getMethod, shouldRetry, ElementWaitDefault);
        }

        /// <summary>
        /// Retrys until action does not throw an exception
        /// </summary>
        /// <param name="action">The action.</param>
        /// <param name="retryForSeconds">The retry for seconds.</param>
        public static void For(Action action, int retryForSeconds)
        {
            var startTime = DateTime.Now;
            while (DateTime.Now.Subtract(startTime).TotalSeconds < retryForSeconds)
            {
                try
                {
                    action();
                    return;
                }
                catch (Exception)
                {
                    Thread.Sleep(500);
                    continue;
                }
            }

            action();
        }

        public static bool For(Func<bool> getMethod, int retryForSeconds)
        {
            return For(getMethod, g => !g, retryForSeconds);
        }

        public static T For<T>(Func<T> getMethod, int retryForSeconds)
        {
            //If T is a value type, by default we should retry if the value is default
            //Reference types will return fase, so our predecate will always pass
            return For(getMethod, IsValueTypeAndDefault, retryForSeconds);
        }

        public static T For<T>(Func<T> getMethod, Predicate<T> shouldRetry, int retryForSeconds)
        {
            var startTime = DateTime.Now;
            T element;
            while (DateTime.Now.Subtract(startTime).TotalSeconds < retryForSeconds)
            {
                try
                {
                    element = getMethod();
                }
                catch (Exception)
                {
                    Thread.Sleep(500);
                    continue;
                }

                //Making it safe for bool and value types and reference types
                if (typeof(T) == typeof(bool) && !shouldRetry(element))
                    return element;

                if (typeof(T) != typeof(bool) &&
                    !IsReferenceTypeAndIsNull(element) &&
                    !shouldRetry(element))
                {
                    return element;
                }

                Thread.Sleep(500);
            }

            element = getMethod();
            return element;
        }

        private static bool IsReferenceTypeAndIsNull<T>(T element)
        {
            return (!(typeof(T).IsValueType) && ReferenceEquals(element, null));
        }

        private static bool IsValueTypeAndDefault<T>(T element)
        {
            return (typeof(T).IsValueType && element.Equals(default(T)));
        }
    }