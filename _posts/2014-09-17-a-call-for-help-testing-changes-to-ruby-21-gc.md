---
layout: post
title: "A Call For Help: Testing Changes to Ruby 2.1.x's GC"
categories: [ruby, heroku]
---

# A Call For Help: Testing Changes to Ruby 2.1.x's GC

## Backstory

There's been some known memory issues when upgrading from Ruby 2.0.0 to Ruby 2.1. Ruby 2.1 introduced a new garbage collector (GC), RGenGC, to improve performance. It traded more memory use for speed. The new GC does less aggressive memory clearing. Doing less work means it's faster.

This sounds great, but on lower memory instances it became an issue since there is less memory available to trade. This [patch](https://github.com/ruby/ruby/commit/a223ff83b07061fe6b7259a72041c4adcc87421b) had been applied to Ruby 2.1 and the hopes are it will solve this [issue](https://bugs.ruby-lang.org/issues/ 9607#note-11). By introducing more frequent GC runs, it will reduce the amount of memory being used.

## Great, How Can I Help?

[@_ko1](https://twitter.com/_ko1)'s patch has already been merged into the Ruby 2.1 branch and queued to be released in Ruby 2.1.3. We (Heroku & ruby-core) would like help verifying this solves the issue at hand with real app data out there.

I've gone ahead and compiled this on Heroku, so it's dead simple to test it. You can run on this newer version of Ruby 2.1.2 by setting this in your Gemfile:

{% highlight ruby %}
ruby "2.1.2", :patchlevel => "216"
{% endhighlight %}

Then after you simply need to deploy this change:

{% highlight sh %}
$ git add Gemfile
$ git commit -m "testing Ruby 2.1.2p216"
$ git push heroku master
{% endhighlight %}

Would you be willing to try this and let us know if it helps with the memory problem?

Once you have data, can you send me a quick note with your results and how it worked out for you: <terence@heroku.com>.
