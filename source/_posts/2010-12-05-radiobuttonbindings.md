---
layout: post
title: RadioButton.IsChecked Bindings
metaTitle: RadioButton.IsChecked Bindings
description: Today I had an issue with binding the IsChecked property of a radio button, and my bindings were being cleared. This is my solution.
revised: 2011-02-19
date: 2010-12-05
categories: [wpf]
migrated: true
comments: true
sharing: true
footer: true
permalink: /radiobuttonbindings/
summary: | 
  

---
Today I had to build a search screen, it had 3 radio buttons for different types of items to search, and each item had very specific search fields.

On my viewmodel I had 3x IsSearchingBlahItems properties. And bound the IsChecked property on my radio buttons to those properties. I then had some stack panels which were bound to the same properties. Here is what this would look like:
<!-- more -->
    <RadioButton IsChecked="{Binding IsSearchingGeneralItems}" /> 
    <RadioButton IsChecked="{Binding IsSearchingJewelleryItems}" /> 
    <RadioButton IsChecked="{Binding IsSearchingWatchItems}" /> 

    <StackPanel Visibility="{Binding IsSearchingGeneralItems, Converter={StaticResource BooleanToVisibilityConverter}}">
        <!--Search Fields-->
    </StackPanel>
    <StackPanel Visibility="{Binding IsSearchingJewelleryItems, Converter={StaticResource BooleanToVisibilityConverter}}">
        <!--Search Fields-->
    </StackPanel>
    <StackPanel Visibility="{Binding IsSearchingWatchItems, Converter={StaticResource BooleanToVisibilityConverter}}">
        <!--Search Fields-->
    </StackPanel>

The behaviour of a radio button group is when you check one of them, it will find all the other RadioButtons in it's group, and Clear the value if it is checked. This will clear your binding =( After going through the different radio buttons all my StackPanel's were visible.

There are many ways to solve this problem, and a bit of googling will show you some of those. I saw using enum's and converters using separate group names for each radiobutton. Then handling checked/unchecked events, then casting datacontext to the viewmodel and setting manually. And a few others.

I went down the road of attached properties, and this is what the end result was:

    <RadioButton Behaviours:RadioButtonBehaviours.RegisterIsChecked="true"
                 Behaviours:RadioButtonBehaviours.IsCheckedBinding="{Binding IsSearchingGeneralItems}" /> 
    <RadioButton Behaviours:RadioButtonBehaviours.RegisterIsChecked="true"
                 Behaviours:RadioButtonBehaviours.IsCheckedBinding="{Binding IsSearchingJewelleryItems}" /> 
    <RadioButton Behaviours:RadioButtonBehaviours.RegisterIsChecked="true"
                 Behaviours:RadioButtonBehaviours.IsCheckedBinding="{Binding IsSearchingWatchItems}" /> 

And here is my behaviour class that does it. Works quite nice and is neat =)

    public static class RadioButtonBehaviours
    {
        public static DependencyProperty RegisterIsCheckedProperty =
            DependencyProperty.RegisterAttached("RegisterIsChecked", typeof(bool), typeof(RadioButtonBehaviours),
                                                new PropertyMetadata(RegisterIsCheckedChanged));

        public static DependencyProperty IsCheckedBindingProperty =
            DependencyProperty.RegisterAttached("IsCheckedBinding", typeof(bool), typeof(RadioButtonBehaviours), 
            new PropertyMetadata(IsCheckedBindingChanged));

        private static void IsCheckedBindingChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            //Transfer update to radio button only on selection
            if ((bool)e.NewValue)
                ((RadioButton) d).IsChecked = true;
        }

        private static void RegisterIsCheckedChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            ((RadioButton)d).Checked += RadioButtonChecked;
            ((RadioButton)d).Unchecked += RadioButtonUnchecked;
            ((RadioButton)d).Unloaded += RadioButtonUnloaded;
        }

        private static void RadioButtonUnloaded(object sender, RoutedEventArgs e)
        {
            ((RadioButton)sender).Checked -= RadioButtonChecked;
            ((RadioButton)sender).Unchecked -= RadioButtonUnchecked;
            ((RadioButton)sender).Unloaded -= RadioButtonUnloaded;
        }

        private static void RadioButtonUnchecked(object sender, RoutedEventArgs e)
        {
            ((RadioButton)sender).SetValue(IsCheckedBindingProperty, false);             
        }

        static void RadioButtonChecked(object sender, RoutedEventArgs e)
        {
             ((RadioButton)sender).SetValue(IsCheckedBindingProperty, true);
        }

        public static void SetIsCheckedBinding(DependencyObject o, bool value)
        {
            o.SetValue(IsCheckedBindingProperty, value);
        }

        public static bool GetIsCheckedBinding(DependencyObject o)
        {
            return (bool)o.GetValue(IsCheckedBindingProperty);
        }

        public static void SetRegisterIsChecked(DependencyObject o, bool value)
        {
            o.SetValue(RegisterIsCheckedProperty, value);
        }

        public static bool GetRegisterIsChecked(DependencyObject o)
        {
            return (bool) o.GetValue(RegisterIsCheckedProperty);
        }
    }