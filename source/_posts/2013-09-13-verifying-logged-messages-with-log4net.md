---
layout: post
title: Verifying logged messages with Log4Net
metaTitle: Verifying logged messages with Log4Net
description: Normally, if you wanted to do this you would try inject ILog. This is a cleaner alternative
revised: 2013-09-13
date: 2013-09-13
categories: [log4net,testing]
migrated: true
comments: true
sharing: true
footer: true
permalink: /verifying-logged-messages-with-log4net/
summary: | 
  Normally, if you wanted to do this you would try inject ILog. This is a cleaner alternative

---
I came across a constructor which looked something like this

    ClassCtor(...., Func<ILog> logFactory) { .. }

The log factory would grab inject a log for the class, but everywhere else in the app used

    ILog log = LogManager.GetLogger(typeof(CLASS));

There must be a better way, something like a scoped appender or something, so I came up with this syntax

    [TestFixture]
    public class RecordTest
    {
        [Test]
        public void DoTest()
        {
            var testing = Log4NetTestHelper.RecordLog(() =>
            {
                var log = LogManager.GetLogger(typeof (RecordTest));
                log.Error("Testing!");
            });
 
            Assert.AreEqual("ERROR - RecordTest | Testing!", testing[0]);
        }
    }

Pretty easy, you record the logs which are logged inside a lambda. This means you don't have to inject logs and modify your code if you decide you want to assert on a logged message

The class to do this is pretty simple

    public static class Log4NetTestHelper
    {
        public static string[] RecordLog(Action action)
        {
            if (!LogManager.GetRepository().Configured)
                BasicConfigurator.Configure();
            var logMessages = new List<string>();
            var root = ((log4net.Repository.Hierarchy.Hierarchy)LogManager.GetRepository()).Root;
            var attachable = root as IAppenderAttachable;
 
            var appender = new MemoryAppender();
            if (attachable != null)
                attachable.AddAppender(appender);
 
			try
			{	        
				action();
			}
			finally
			{
				var loggingEvents = appender.GetEvents();
				foreach (var loggingEvent in loggingEvents)
				{
					var stringWriter = new StringWriter();
					loggingEvent.WriteRenderedMessage(stringWriter);
					logMessages.Add(string.Format("{0} - {1} | {2}", loggingEvent.Level.DisplayName, loggingEvent.LoggerName, stringWriter.ToString()));
				}
				if (attachable != null)
					attachable.RemoveAppender(appender);
 
			}
			
            return logMessages.ToArray();
        }
    }

Enjoy!