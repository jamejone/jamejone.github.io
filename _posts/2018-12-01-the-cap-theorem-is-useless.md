---
layout: post
title: The CAP theorem is useless for enterprise software
published: false
---

*A practical guide to achieving scalable, available and always-consistent web services using 
commodity hardware and eventually-consistent technology.*

The CAP theorem provides a comprehensive explanation of the tradeoffs involved with having an atomic piece of information exist in two places at once. In not so many words, it claims all distributed systems must make a compromise, either:

* Provide potentially stale versions of atomic information, or...
* Utilize a distributed locking mechanism to limit access to the system while the system synchronizes the copies of atomic information.

While true at a theoretical level, it crucially misses the mark by failing to propose a useful approach to cope with the potential for stale state in a distributed system. We challenge the assertion that “most real-world systems are forced to settle with returning ‘most of the data, most of the time.’” On the contrary, non-realtime enterprise systems are in fact never forced into such a circumstance and can in fact predictably return all of the data, every time.

### Intended audience

This guide is for system designers who want to design a scalable system in the collaborative domain. This guide isn't for you if:

*You're developing a system which lacks a collaborative aspect*. In this case horizontal scalability should be quite simple. Some examples are note-taking apps, exercise trackers, web pages with static information, most Excel-type apps.

*You're developing a system which will only ever have a handful of users*. In this case you'll needlessly complicate your system with the patterns presentend here. Since scalability isn't a concern you can get away with a lot by utilizing the transaction capabilities of your database along with some concurrency checks. Line-of-business apps fall into this category. These kinds of apps are extremely common (and valuable!).

*You're designing a realtime system*, which is a system which needs to make decisions within hard deadlines. This is simply a trade-off of the design presented here (and a useful one at that). Examples of this type of system are stop lights, air traffic control systems, train management, high-speed trading. 

It's common to mistakenly believe you're dealing with a realtime domain - *"Accounting needs this report by 9AM or they'll shit a brick!"* If the system requires an extra 20% processing time because trading volumes were 20% higher that day, business users will generally be understanding. What they *don't* like is not being able to quickly and cheaply fix the issue by simply provisioning more commodity hardware (which is what you'll get if you design a real-time system!)

The problem with realtime is that it exponentially increases the complexity of the system as it needs to be able to make decisions without all of the required information -- information which will be available in just a few moments' time. In most cases the value proposition for this massive increase in complexity simply just isn't there.

A good way to know whether you have an accidental realtime system on your hands is:

* A batch process kicks off at a pre-determined time of day, and...
* ... the batch process depends on another process completing on-time.

Avoid this pattern of placing artificial hard deadlines on the system. If processes need to occur in-order then use a queue, eventing pattern or callback mechanism.

Here are some rules of thumb: If you're not using FPGAs and advanced hardware acceleration, you're probably not realtime. If you're writing a web app on the cloud, you're almost certainly not realtime!

Assuming you're still interested, without further ado...

### Defining the problem

Let's use the example of an online store. The crux of the collaborative problem here is:

1. One of the most important requirements is that our system be always available to accept purchase orders. In order to do this, we'll need to accept orders across many servers.
2. Orders which are placed on different servers may be competing with one another (i.e. two customers trying to order a product for which insufficient quantity of product exists to satisfy both orders).
3. Decisions about who wins and loses must made once we have established all the facts in one place and order them chronologically. (i.e. customer 1 placed their order at exactly 8:52:32 PM, then customer 2 placed their order 3 seconds later at 8:52:35 PM).
4. The process of assembling a chronological history of events will require some amount of time which will vary depending on how much history there is to assemble. In other words, a sudden influx of purchase orders across many servers will take longer to process than a slow trickle.

### Slow people, less slow machines

Information systems which deal in the collaborative domain almost always solve the collaborative problem with a two-step process:

1. Declare intent to change the state of a shared resource (e.g. place an order for the purchase of a product or cancel a purchase).
2. Provide an asynchronous workflow which actually performs the state change, resolving conflicts if necessary (e.g. fulfill a purchase order or submit a back order).

This is how information systems have worked for hundreds of years. Yes, even before computers. Back when the only computers were, well, people who compute. Today we have machines that can do this work for us, but for some reason we still use the old two-step process.

Why is this? In the Information Age, do we still need to use this old workflow? Can't we just instantaneously solve the collaborative problem in a single step?

The short answer is no, but we can come pretty darn close!

### Exploring potential solutions

*Fulfil orders after-hours*

If you can afford to close your online business in order to fulfil orders then by all means do so. You'll certainly cut down on the complexity of the system. If you're curious what keeping your business open 24/7 will cost you, read on...

*Dedicate one server per product sold* 

This eliminates the need to solve the collaborative problem across multiple machines. And this would indeed be an compelling option if it did not needlessly restrict our capacity to sell a given product. It would be a shame to have so many machines sitting idle while we struggle to move our hottest product.

*Queue orders as they enter the system and process them later*

This is actually a fairly effective practice. Queues are crazy scalable because they spare you from worrying about resolving the collaboration problem until you consume from the queue. If you only have one queue consumer, congratulations, you've solved the collaboration problem! And if you temporarily can't write to the database for whatever reason, you can rest assured orders will still flow into the queue.

The problem with this type of system lies in its complexity and fragility. You're now dealing with at least one or possibly many queues, which is a new type of infrastructure to learn, implement and manage. Your disaster recovery plan becomes way more complicated as losing a queue would require refilling the replacement queue. Synchronizing the queue state and the database state will be an ongoing concern.

What alternatives remain?

### Meet the new system, same as the old system

Huh? You were hoping for Star Trek-level advanced computational theory? Well, I'm sorry to disappoint you. We're going to solve this problem using some old fashioned techniques which have existed for centuries!

First, partition purchase orders on ID (or geolocation if replica sets distributed globally) so they're randomly distributed among servers. Partition the inventory ledger on product ID.

### Wow, that was a lot. Why does this matter?

It matters because computers are slow. They were never very fast to begin with. And they're only getting slower.

Mostly because computers were never that fast to begin with. Processing pipelines can only be optimized so much. Plain and simple.

On top of that, we're reaching theoretical limits for how fast computers can go. Each attempt to increase the clock speed of a CPU results in more and more outsized costs. CPU designers are instead opting for more processing cores in lieu of clock speed. Even further, clock speeds are in fact *slowing down* as system designers adopt value-oriented "commodity" hardware designs.

In a sense, it's useful to think of computers as slow (and getting slower).

---









Scalable information systems almost always provide separate, efficient mechanisms for receiving and outputting data. This is because searching through a bunch of information takes a lot of effort, even for a computer.

If this weren't the case, then the business of inventory management would be much simpler. Any time you want to know the quantity of a given product on-hand, the system would instantaneously roll up every order involving that product.

However even in this idealistic scenario, this would of course mean that your inventory amount is immediately out-of-date the moment you ask for it. People generally have an intuitive sense that the information they possess reflects the state of the world at some point in the past. An item that's in-stock may in fact be out-of-stock. Anyone that's experienced their order not being fulfilled understands this phenomenon. This is the price we pay for systems that operate globally and provide instantaneous feedback.

An item that goes out-of-stock does not immediately reflect itself as such.

Someone (or maybe even several people) may have placed an order for the last item in-stock. To the dismay of the others, only one order will be fulfilled. This is the price we pay for systems that operate globally and provide instantaneous feedback.






If this weren’t the case, then the business of banking would be much simpler. Any time you want to know your account balance, your bank would instantaneously roll up every transaction you've ever made on the account.

Even in this idealistic scenario, this of course would mean that your bank balance is immediately out-of-date the moment after you ask for it. People generally have an intuitive sense that the information they possess reflects the state of the world at some point in the past. And perhaps even the source of their information was reasonably out-of-date.

This is the price we pay for systems that operate globally and provide instantenous feedback.









This wouldn't be very different than a nation conducting a census every time someone asked "How many people live here?" In most cases it doesn't matter whether how old information is, just as long as we're confident that the facts presented to us represent the state of the world as it existed *at a single point in time*.

This is why when 









A common way of describing this type of implementation is having a “write-optimized model” and a “read-optimized model”. The write-optimized model is optimized for ingesting data as quickly as possible and the read-optimized the opposite. The frequency at which the read-model is updated to reflect the write-model is left up to the system designer. Some systems attempt to instantaneously keep the two in-sync by preventing read and write operations from happening in parallel while the models are being synchronized (relational databases are a familiar example of this type of system). This is a comforting illusion as it provides the perception that you’re always reading the present and correct state of the system. Unfortunately, this feature comes at a cost of lower system availability and lower overall throughput as the state of the system changes. Thus, this type of guarantee is only available for static data or small data sets that can fit on a single computer.

Nearly all scalable and available systems allow some lag between receiving data and being able to provide an aggregated output of that data. Google certainly can’t instantaneously update their search engine each time someone updates their web page, nor can they block someone from updating their web page until they update their index. People generally have an intuitive sense that aggregated information is lagged from reality.

This phenomenon is not so different than paper-driven, manual information systems of the past. The biggest difference is that aggregation can be performed much more quickly today with computers. The aggregation still nonetheless occurs within a dimension of time, so all the traditional methods for resolving conflicts and applying business rules still apply. In a sense, computer-driven systems aren’t so different than the paper-driven systems they’ve replaced.

Take, for example, the arduous process of compiling a phone book in the 19th century. The telephone company would begin by retrieving the latest copy of each customer’s information from their file. Customers could continue submitting updates to their information by submitting new versions of their contact information. They would do this simply by filling out a new paper form and handing it to the proper person at the telephone company, who would in turn immediately file it away or forward it to a more centralized storage location.

Any new submissions after the phone company already started drafting the new phone book won’t be considered until next year’s phone book. Why? Well, obviously it would be too much work to start over each time a customer updates their information. It also wouldn’t make much sense to start using the customer’s new information in the middle of compiling the new book. The customer’s old information would appear in the last name lookup, and their new information in the reverse phone number lookup. This surely won’t do. In this respect there exists a business-consistency boundary.

The filing cabinet provides a versioned, immutable, write-optimized model of customer information. It’s versioned in the sense that each time the customer updates their information they’re submitting a brand new form that they filled out and it goes straight into the filing cabinet. The old version of their information is allowed to coexist alongside the new version of their information. And the versions of their information are affixed with a date so we always know which one is newer.

The phone book provides a business-consistent, read-optimized model. You can quickly look anyone up according to their first name, last name, phone number or address. Sure, it might be slightly out-of-date, but people generally have an intuitive expectation that aggregated information is out-of-date and know to look at the front cover for the date the book was made. Not bad for an 1800’s information system!
