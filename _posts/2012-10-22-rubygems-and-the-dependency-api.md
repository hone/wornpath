---
layout: post
title: Rubygems and the Dependency API
category: bundler heroku
---
## Issues :(
If you've ever done anything in ruby, you've probably used [rubygems.org](http://rubygems.org) to search or install your favorite gem like [rails](http://rubyonrails.org). Our beloved gem service has been having issues lately and went down hard on Wednesday (10/17). We deduced that the Dependency API used by Bundler 1.1+ was the culprit. It was a bit too successful and caused too much load on the app server. The service was brought back thanks to work by [Nick Quaranto](http://twitter.com/qrush) and [Evan Phoenix](http://twitter.com/evanphx). Thanks guys! Unfortunately, the Dependency API had to be disabled to get rubygems.org back up.

## Forging a Path Forward
With the API down, one of the ideas we tossed around in Bundler core was extracting the API as a separate app to reduce the burden on rubygems.org. I paired with [Daniel Farina](https://twitter.com/danfarina) from [Heroku](http://heroku.com)'s data team to build the prototype. We built a simple [sinatra](http://sinatrarb.com) app that just responds to the `/api/v1/dependencies` endpoint and redirects everything else to rubygems.org. The second important piece is keeping the gem metadata up to date. This is done by just fetching the modern indexes from Cloudfront and diff'ng against the local postgres cache to see what new gems have been added and gems to be yanked. The new gem data can then be fetched from the `.gemspec.rz` files from Cloudfront to get the dependency information.

A pow wow was held on Friday (10/19) to figure out steps moving forward to get the API back up as soon as possible. Everyone agrees this is a detriment to the community as whole. You can read the [notes](https://gist.github.com/3921560) or watch the [youtube video](http://youtu.be/z73uiWKdJhw). We agreed that the best option at the moment was to go ahead with the sinatra app and host it on Heroku. We'll be work on load testing it this week so it can be rolled out.

## Helping Out!
You can help in testing the service to make sure it's battle tested and works like expected. You can do this by setting up your Gemfile to point to the Heroku app I've built. Then please reach out to me with feedback on twitter, [@hone02](http://twitter.com/hone02) or as a comment below.

Gemfile:
{% highlight ruby %}
source "http://bundler-api.herokuapp.com"

gem 'rack'
{% endhighlight %}

The source code for the app lives on github, <http://github.com/rubygems/bundler-api>. Please feel free to code review it. I'll try doing a separate blog post going over the code, especially the syncing code in detail.

## Other Information
* [Mailing list thread](https://groups.google.com/forum/?fromgroups=#!topic/rubygems-org/-xlwOYV9Bgw)
* [History on Fetching the Source Index](http://robots.thoughtbot.com/post/2729333530/fetching-source-index-for-http-rubygems-org)
