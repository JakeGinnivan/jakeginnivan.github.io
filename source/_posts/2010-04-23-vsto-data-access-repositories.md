---
layout: post
title: Outlook Items, Repositories & Data Access
metaTitle: Outlook Items, Repositories & Data Access
description: Data Access in VSTO can be hard, I will discuss some ways to avoid performance issues, leaky abstractions and make it much easier.
revised: 2011-02-19
date: 2010-04-23
categories: [VSTO]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-data-access-repositories/
summary: | 
  

---
If we want to do anything interesting with VSTO and Outlook we will have to access data from Outlook. That will normally be Contacts, or Appointments or Emails? When you do this you will probably leak items, then Outlook will stop behaving normally, you can get ghost inspectors (click save & close, and the window stays open, but ribbon is grayed out), or errors when you try to edit a item through the Outlook UI. There can be many side effects, most are not fun to track down the cause.
<!-- more -->
<h1>Repository Pattern</h1>

This seems as good a place as any to cover the repository pattern, it is REALLY important if you want a stable VSTO app. If you are unsure what it is then have a read of [http://martinfowler.com/eaaCatalog/repository.html][1].

Essentially my repository interfaces will expose methods like GetAppointments(), SaveAppointment(), NewAppointment() etc. I will use repositories rather than interacting with the Outlook session directly. This prevents me accidently leaking COM objects.

To achieve a non leaking high performance repository no matter what thread you call from we have to use a few patterns and Automapper to put it all together. Here is a class diagram of the end goal.

![Outlook Event Repository][2]

Notice that my repository returns a basic CLR interface, not appointment items.

FacebookEventAdapter – uses the adapter pattern to adapt the ugly and HUGE _AppointmentItem interface to something that is useful to us. It is essentially a wrapper class that adapts the _AppointmentItem interface to our own interface.

![Outlook event interface][3]

The next step is we do not want our repository to leak. For the moment trust me that we do not want to leak COM objects. I will cover why it is bad in a later post.

This is the point that AutoMapper is essential:

    public IList<IOutlookFacebookEvent> GetEvents()
    {
        var events = new List<IOutlookFacebookEvent>();
        using (var calendar = _session.GetDefaultFolder(OlDefaultFolders.olFolderCalendar).WithComCleanup())
        using (var items = calendar.Resource.Items.WithComCleanup())
        {
            events.AddRange(
                items.Resource
                    .ComLinq<AppointmentItem>()
                    .Select(appointment => new FacebookEventAdapter(appointment))
                    .Select(adapter => Mapper.Map(adapter, new OutlookFacebookEvent(adapter.RsvpStatus, adapter.EntryId)))
                    .Where(e => e.EventId != -1));
        }

        return events;
    }

Ignore the WithComCleanup and ComLinq extension methods for now, I will cover them in a later post (or check out Outlook.Utility which contains those extensions). They simply make sure that the COM objects are cleanly disposed of and stop COM object leaks.

What is important is that we wrap our appointmentItem in the adapter, then we map the adapter to our CLR implmentation of our interface. And there is only 1 line of code need to make the mapping work:

    Mapper.CreateMap<FacebookEventAdapter, OutlookFacebookEvent>();

Here is a step by step description of what is happening:

 1. Get the Calendar folder from Outlook
 2. Get all Items in the collection
 3. ComLinq exposes a custom IEnumerable<T> which returns a custom IEnumerator<T> which releases the COM objects as soon as the enumerator moves onto the next item. And has the added advantage that it gives is Linq to Object support on the COM collection =)
 4. For each _AppointmentItem in the calendar, wrap it in a FacebookEventAdapter
 5. Then Map the adapter to a new CLR implementation of our IOutlookFacebookItem interface.
 6. Filter any items that are not Facebook events.
 7. Add the resulting items to the events list and return it.

The end result is that we return clean CLR objects which have no COM references and all the COM objects that were brought into .NET have been released before we return.

<h1>Dispatching Repository</h1>

As I said before Outlook runs in a STA. This means that if we call our repository from a background thread (which you will do if you are doing synchronisation or some other expensive operation) every time we interact with the Outlook Interop API we cause the call to be marshalled across threads.

The solution – a repository [decorator][4] which makes any repository calls on the Outlook STA thread, so we have only a single call that has to be marshalled across threads, then all Outlook calls will already be in the STA so no marshalling required.

I setup a small test app to test my theory that I would actually get performance gains by doing this, and I was amazed at the results.

**Mapping ~1000 Appointments to CLR objects**

> Background repository mapped in ~10 seconds <br />
> Dispatched repository mapped in ~4 seconds

**Mapping 16 Appointments to CLR objects**

 > Background repository mapped in ~1.6 seconds <br />
 > Dispatched repository mapped in ~0.6 seconds

So large or small data sets, there is still a massive performance gain by invoking the call on the STA thread before doing the COM interop work. Here is the code to do that (In production more error handling is probably needed):

    public IList<IOutlookFacebookEvent> GetEvents()
    {
        var getEvents = ((Func<IList<IOutlookFacebookEvent>>)(() => _outlookEventRepository.GetEvents()));

        return (IList<IOutlookFacebookEvent>)_outlookStaDispatcher.Invoke(getEvents);
    }

<h1>Wrapping Up</h1>
So we have used quite a few patterns together to achieve a very solid repository that performs well from any thread you call it from. We used the Repository Pattern, the Adapter Pattern and the Decorator Pattern to make it all work.


  [1]: http://martinfowler.com/eaaCatalog/repository.html
  [2]: /get/screenshots/OutlookEventRepository.png
  [3]: /get/screenshots/IOutlookFacebookEvent.png
  [4]: http://en.wikipedia.org/wiki/Decorator_pattern