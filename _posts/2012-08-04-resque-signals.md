---
layout: post
title: Resque Signals
category: resque
---
[Resque](http://github.com/defunkt/resque) is one of the major queuing libraries in Ruby. I've taken over maintenance of the project since January 2012. Over the last few months, we've been working on ironing out the last few major things in [Resque 1.x](http://github.com/defunkt/resque/tree/1-x-stable). One of the last holdouts is the way signals are handled.

## Problem

For signal handling, Resque borrows from nginx (from the [README](https://github.com/defunkt/resque/blob/1-x-stable/README.markdown#signals)):

* [QUIT](http://en.wikipedia.org/wiki/SIGQUIT) - Wait for child to finish processing then exit
* [TERM](http://en.wikipedia.org/wiki/SIGTERM) / INT - Immediately kill child then exit

The current implementation differs from standard UNIX philosophy on this. Traditionally, you use SIGQUIT to exit immediately and core dump. SIGTERM is then used to ask a process to terimnate nicely giving it a chance to clean itself up.

If you treat the resque worker process like any other unix process, you'd probably send a SIGTERM to the worker to ask it to shutdown. Historically, this will [immediately kill the child using SIGKILL](https://github.com/defunkt/resque/blob/0286e695402179661552897392c7221e23350181/lib/resque/worker.rb#L308). Just a note, the child process is where the job work is being performed. If you're in the middle of a long running job, you'll have no way to run any clean up code. At Heroku, we use resque to handle background work for dealing with [database backups](https://devcenter.heroku.com/articles/pgbackups). This signal handling won't work, since we need to mark this job as failed in our databases, so the user knows the backup never finished.

## Solution

In Resque 1.22.0, we've addressed this issue by adding [a new signal handling path](https://github.com/defunkt/resque/blob/de26a891253be9f9642685918cb0e81f16ff992c/lib/resque/worker.rb#L349-366) that will be the default in Resque 2. The old way is deprecated as of this release. Since, we're held to semantic versioning, the old way is still the default. In order to opt-in you just need to set the environment variable `TERM_CHILD`. The changes include a new flow as well as reverting the trap from the parent in the child, so the child can now receive signals properly.

{% highlight sh %}
$ TERM_CHILD=1 QUEUES=* rake resque:work
{% endhighlight %}

With these changes the new signal flow is the following:

* you send SIGTERM to the worker
* the parent worker sends SIGTERM to the child process
* the parent worker waits for a period of time for the job to exit successfully
* if the child is still running, the parent worker sends SIGKILL to the child
* the parent worker then exits

The advantage of this is it allows the job to be notified of being killed. In the database example above, we can now write our job this way.

{% highlight ruby %}
require 'resque/errors'

class DatabaseBackupJob
  def self.perform
    # do backup work
  rescue Resque::TermException
    # write failure to database
  end
end
{% endhighlight %}

Rescuing `Resque::TermException` allows us to have a block of code that executes during that "wait period" for cleanup purposes. If you don't want to `require 'resque/errors'` in your job, you can just rescue the traditional `SignalException` that ruby throws.

{% highlight ruby %}
class DatabaseBackupJob
  def self.perform
    # do backup work
  rescue SignalException
    # wiret failure to database
  end
end
{% endhighlight %}

The "wait period" is customizable with the environment variable `RESQUE_TERM_TIMEOUT` and its unit is in seconds. By default, we use 4 seconds.

{% highlight sh %}
$ TERM_CHILD=1 RESQUE_TERM_TIMEOUT=10 QUEUES=* rake resque:work
{% endhighlight %}

## Example

So let's a look at how this works. Here's a really simple job I'm using:

{% highlight ruby %}
require 'resque/errors'

class SleepJob
  @queue = :test

  def self.perform(time = 100)
    sleep(time)
  rescue Resque::TermException
    sleep(2)
    puts "omg job cleaned up!!!!"
  end
end
{% endhighlight %}

### Deprecated Signal Handling

This is the old way we're handling signals. Here's the command to run this on Resque 1.22.0.

{% highlight sh %}
env VVERBOSE=1 QUEUE=* bundle exec rake resque:work
{% endhighlight %}

With the log output we can see that when we send SIGTERM, the resque just kills the child immediately with no remorse.

    20:30:08 worker.1     | started with pid 18903
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: Starting worker hone-x220:18903:*
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: Registered signals
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: Checking archives
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: Checking test
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: Checking timeline
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: Sleeping for 5.0 seconds
    20:30:08 worker.1     | ** [20:30:08 2012-08-15] 18903: resque-1.21.0: Waiting for *
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18903: Checking archives
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18903: Checking test
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18903: Found job on test
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18903: got: (Job{test} | SleepJob | [30])
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18903: resque-1.21.0: Forked 18946 at 1345080613
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18946: resque-1.21.0: Processing test since 1345080613
    > Process.kill("TERM", 18903)
    20:30:23 worker.1     | ** [20:30:23 2012-08-15] 18903: Exiting...
    20:30:23 worker.1     | ** [20:30:23 2012-08-15] 18903: Killing child at 18946
    20:30:23 worker.1     |   PID S
    20:30:23 worker.1     | 18946 S
    20:30:23 worker.1     | process terminated

### New Signal Handling Cleaning Up

This is the case where we use the new signal code.

{% highlight sh %}
env TERM_CHILD=1 VVERBOSE=1 QUEUE=* bundle exec rake resque:work
{% endhighlight %}

In the logs we can see it actually executes the cleanup code and prints "omg job cleaned up!!!!".

    20:36:10 worker.1     | started with pid 21301
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: Starting worker hone-x220:21301:*
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: Registered signals
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: Checking archives
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: Checking test
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: Found job on test
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: got: (Job{test} | SleepJob | [30])
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: resque-1.21.0: Forked 21318 at 1345080971
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21318: resque-1.21.0: Processing test since 1345080971
    20:36:20 worker.1     | ** [20:36:20 2012-08-15] 21301: Exiting...
    > Process.kill("TERM", 21301)
    20:36:20 worker.1     | ** [20:36:20 2012-08-15] 21301: Sending TERM signal to child 21318
    20:36:25 worker.1     | omg job cleaned up!!!!
    20:36:25 worker.1     | ** [20:36:25 2012-08-15] 21318: done: (Job{test} | SleepJob | [30])
    20:36:25 worker.1     | process terminated

### New Signal Handling Timing Out

This case is where we use the new logic, but the cleanup code takes longer than the time allowed. We simulate this by sleeping longer than the allotted time.

{% highlight sh %}
env RESQUE_TERM_TIMEOUT=1 TERM_CHILD=1 VVERBOSE=1 QUEUE=* bundle exec rake resque:work
{% endhighlight %}

In the logs we can see the "omg job cleaned up!!!!" isn't printed.

    20:26:26 worker.1     | started with pid 17395
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: Starting worker hone-x220:17395:*
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: Registered signals
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: Checking archives
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: Checking test
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: Found job on test
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: got: (Job{test} | SleepJob | [30])
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: resque-1.21.0: Forked 17412 at 1345080386
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17412: resque-1.21.0: Processing test since 1345080386
    20:26:49 worker.1     | ** [20:26:49 2012-08-15] 17395: Exiting...
    > Process.kill("TERM", 17395)
    20:26:49 worker.1     | ** [20:26:49 2012-08-15] 17395: Sending TERM signal to child 17412
    20:26:53 worker.1     | ** [20:26:53 2012-08-15] 17395: Sending KILL signal to child 17412
    20:26:53 worker.1     | process terminated

I believe this new way of signal handling is less broken and superior to the past implementation. It fixes both the child process not handling signals properly and provides a clean way to add clean up code to your job. Again, this is the path forward on how processes will be handled in Resque. I would like to call out a special thanks to [Jonathan Dance](http://github.com/wuputah), [Ryan Biesemeyer](https://github.com/yaauie), and [Aaron Patterson](http://github.com/tenderlove) for design discussion and work on this feature.
