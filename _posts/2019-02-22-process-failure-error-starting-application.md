---
layout: post
title: Bug fix - HTTP Error 502.5 - Process Failure
published: true
---

I encountered a weird error today attempting to run a .NET Core application hosted in IIS. The error was:

![An error occurred while starting the application.]({{ site.baseurl }}/public/2019-02-22-process-failure-error-starting-application/error-starting-application.jpg)

> *An error occurred while starting the application.*

Well that's a little vague. It worked fine in IIS Express. So I decided to restart my computer because that's what I do when I need some time to think, and every once in a while it helps. In this case it actually did help by giving a different error message the first time I visit the page after a fresh restart:

![HTTP Error 502.5 - Process Failure]({{ site.baseurl }}/public/2019-02-22-process-failure-error-starting-application/process-failure.jpg)

> HTTP Error 502.5 - Process Failure
> 
> Common causes of this issue:
> - The application process failed to start
> - The application process started but then stopped
> - The application process started but failed to listen on the configured port

It occurred to me that since this is an ASP.NET Core application that the ASP.NET Core module for IIS is attempting to launch an executable, which I can also execute at will. Running the executable directly gives me a clear indication of the problem on the console output -- there's an uncaught exception being thrown in Startup.cs:

![Incriminating stack trace]({{ site.baseurl }}/public/2019-02-22-process-failure-error-starting-application/stack-trace.jpg)

There's the problem! A missing config variable. I love when I don't need to scour the internet with cryptic error codes. ASP.NET Core told me exactly what the issue was.