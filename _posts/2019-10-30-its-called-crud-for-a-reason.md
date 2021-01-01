---
layout: post
title: It's called 'CRUD' for a reason
published: true
description: Most modern ways of doing things can be traced back to a historical problem that was solved using the path of least resistance.
---

![Aww, did somebody get addicted to RDBMS transactions?]({{ site.baseurl }}/public/transaction-addicted.jpg)

> *"To err is human, but to really foul things up you need a computer."*
>
> -Paul R. Ehrlich

Many business processes that exist today were born out of old fashioned paper/form-driven processes. This is because paper was invented before computers were invented. Had computers been invented first, you might imagine a completely different world in which you place an order online to buy something and your browser spins until the package arrives on your doorstep.

I'm kidding, sort of.

Many enterprise software systems are less exaggerated versions of this. In some cases I've seen 'submit' buttons routinely keep the user waiting as long as several minutes for the underlying transaction to complete.

Why has the bar fallen this low? And how can we fix it? In this post we'll dive into exactly that.

### History doesn't repeat itself, but it often rhymes

Most modern ways of doing things can be traced back to a historical problem that was solved using the path of least resistance. From the icons on your phone to the arrangement of the keys on your keyboard, these enduring vestiges of the past surround us.

Back when the world was run on paper, modifying a piece of information might have meant moving a giant, heavy filing cabinet. It made much more sense to grab a fresh form and fill it out, with the intent of replacing the original form later on. This is a very intuitive practice because the cabinet is heavy and slow, and there aren't significant consequences for updating original form at your convenience. And if finance wants to do a quarter-end process that takes a few days and depends on these forms, you can let them do their thing unbothered by everyone wanting to update their information. Many processes like this exist because they feel natural.

Fast forward to the digital age, our first (and correct) assumption is that computers are fast. To assume otherwise would be a missed opportunity to design a simpler system that performs adequately. In the industry this dilemma is known as "premature optimization." In the typical lifecycle of a software system, the first step is to build a rudimentary proof-of-concept that's gradually modified to accommodate the increased demands placed upon it.

The simplest kind of computer information system is a CRUD-like design. CRUD stands for Create/Read/Update/Delete, which refers to the way data is stored on the system. In this kind of system, every entity in the system reflects its most recent write. If the need arises to keep multiple entities in a consistent state or capture the state of the system at a single point in time, a dark magic known as ACID transactions are employed. A CRUD design is effectively the same as the paper-driven process except your data entry person is The Flash and is capable of immediately updating information upon arrival and can also generate any kind of report you need, when you need it.

The unfortunate reality is that even The Flash has his limits. CPU speeds haven't budged in decades and data requirements continue to grow exponentially. And as it turns out, [ACID transactions aren't magic](https://pdfs.semanticscholar.org/136c/3bb91cb7984fa7add24d197f1056465e0975.pdf). They typically rely on a mix of locks (which reduce system availability) or optimistic concurrency (which reduces reliability and consistency). These effects are ever more pronounced as the system scales. [Mutable state, scalability and consistency are sworn enemies](https://en.wikipedia.org/wiki/CAP_theorem).

Like a thin layer of filth (crud?) that slowly accumulates on your lenses, it can be difficult to notice that you've gradually moved the goalposts on the responsiveness of the system. It's tempting to blame the CAP theorem for your woes and settle for investing weeks of developer time into regaining marginal amounts of lost ground. Any progress can seem impressive when you're desperate enough. But fear not, there's solution to all of this, you just haven't seen it in a very, very long time.

### Meet the new process, same as the old process

It would be easy for me to throw software developers under the bus for not championing a solution to this problem. The reality is that business process owners are equally culpable. After all, early business process owners were able to conquer these issues using the slowest and most error-prone computers of all -- human beings.

The major difference between the slow, paper-driven processes of yore and the "fast," computer-driven processes of today is something called "temporality." It's this idea that you can have multiple versions of a piece of information, each valid as of their respective points in time. By incorporating the dimension of time, the system can recall what its exact state was at any given point in time.

Early information systems possessed elements of temporal systems by virtue of the fact that the easiest thing to do was to version the data stored within them. By intentionally utilizing this concept in digital information systems, we can achieve a whole new level of scalability, responsiveness, reliability, testability and simplicity. I can't wait to show you how easy it can be to design this sort of system. Stay tuned.

**See the follow-up blog post:** [Temporal repository implementation using MongoDB and ASP.NET]({{ site.baseurl }}/2019/11/14/temporal-repository-implementation-using-mongodb-and-aspnet-core/)