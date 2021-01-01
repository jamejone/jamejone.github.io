---
layout: post
title: Real-time by accident
published: true
---

![If you could come in on Saturday, that would be great.]({{ site.baseurl }}/public/office-space-meme-milton.jpg)

Wednesday, 8:55 AM.

You get settled into your chair and log in. Time to get some work done.

...

9:35 AM.

You hear someone tapping on your doorway. 

"The 8:00 AM daily batch job failed run. Can you look into it?"

9:51 AM.

You discover what happened. The 4:00 AM nightly batch job took longer than usual to run. Your excellent logging indicates it was processing an usually large set of data. In fact, it's still running right now and it's almost complete. You fire off an email to let the concerned know what's going on and that you'll restart the 8:00 AM job once the 4:00 AM job is done.

Crap! You're late for the 10:00 AM standup.

...

11:05 AM.

You finally arrive back at your desk, quickly verify the 4:00 AM job finally finished, manually kick off the job that was supposed to run at 8:00 AM, and send another email letting the user know when the job finishes. It usually only takes 30 minutes or so. Try to get some work done in the meantime.

11:35 AM.

Crap, the job's still running.

11:40 AM.

Still running...

11:45 AM.

The job's done! Fire off a third email letting them know the issue's resolved.

What do you know, it's lunch time! Where'd the morning go?

### Here's where it went

You spent it babysitting a faux-real-time system.

It's real-time in the sense that processing deadlines are imposed on the system. In our example, the 4:00 AM job has a [worst-case execution time](https://en.wikipedia.org/wiki/Worst-case_execution_time) of 4 hours. It's a [hard deadline](https://en.wikipedia.org/wiki/Real-time_computing#Criteria_for_real-time_computing) in the sense that the system fails if the jobs don't complete on-time. It's 'faux' in the sense that these constraints are unnecessary and engender a lot of serious drawbacks.

Scheduling batch jobs this way lends itself to tedious management of issues arrising from the inherent race conditions involved. Additionally, it's difficult to determine the dependencies between the jobs since they appear more coincidental than explicit. This sort of knowledge is usually known by everyone but never defined anywhere. "Everyone knows the 4:00 AM job needs to be done before the 8:00 AM job starts!" And worst of all, these patterns tend to fail you just when you need them to work the most.

This pattern is usually borne out of a few reasons:

* An attempt to sequence the jobs in a decoupled way,
* The availability of 'cron'-like job scheduling tools, and 
* The apparent lack of equivalently-available and simple alternatives.

So how can we get ourselves out of this mess? Are there any alternatives?

![The only winning move is not to play.]({{ site.baseurl }}/public/only-winning-move.gif)

### Potential solutions

#### Mutex locks (don't do this)

One solution I've seen at a major international company was to keep the existing scheduling system and augment it with mutex locks. Specifically:

1. Create a dependency graph for every job.
2. When each job starts, attempt to obtain a lock on all dependencies that job depends on (including the job itself).
* In case of failure to obtain all the locks, release the locks and try again in several minutes.
* If all locks were successfuly obtained, start the job.
3. When the job finishes, release the locks that were obtained in order to start the job.

Although this approach worked in the short-term, it was fraught with issues. If a job goes longer than usual and causes other jobs to enter a retry state, you never know which of the retrying jobs will go next. Oftentimes you want them to go in chronological order, but retry logic doesn't naturally enable this sort of behavior. You'd need to tack on some additional logic which looks at the other jobs and this quickly gets very convoluted.

There are other issues such as deadlocks and lost locks. Just stay away from this pattern.

#### Event sourcing/CQRS/Job queueing

Much smarter people than me have written about these patterns in vast detail, so I won't waste time rehashing their work. In essence, we want to remove the timing-based scheduling and replace it with chaining or queueing. The end of one job should indirectly trigger the next job, ideally without the jobs knowing about each other.

This is roughly how it would look:

1. At 4:00 AM, a command is submitted to the command queue to start Job 1.
2. An agent picks up Job 1 and starts working on it. Once it ends the agent creates a "Job 1 completed" event.
3. The "Job 1 completed" event is translated into a command to start Job 2 and the command is subsequently submitted to the command queue.
4. An agent picks up the Job 2 command and starts working on it.

At its heart we're literally linking the jobs together. In other words, when one job ends another quickly starts after it. The dependency is explicitly defined in code (a "Job 1 completed" event *causes* a Job 2 command to be submitted).

Sure, it's possible that one job severely delays the other jobs, but in enterprise system that's generally okay.

As the adage goes, better late than never.