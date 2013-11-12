---
layout: post
title: Goodbye Bundler 1.4, Hello Bundler 1.5
category: bundler
---

People [have been asking](https://twitter.com/sikachu/status/399944484961918976) why we jumped from a 1.4 release candidate to a 1.5 one without ever releasing Bundler 1.4.0.
The Bundler core team takes releases very seriously and tries to be an example of thoughtful release practices to the Ruby community. Since we follow standard [releases candidate policies](http://en.wikipedia.org/wiki/Software_release_life_cycle#Release_candidate), release candidates are automatically under a feature freeze. Each release candidate gets roughly two weeks of testing, and any significant bugfixes result in a new release candidate and the timer starting over. The exact code from the first release candidate with no significant bugs will be the code released as the final version.

When we released [Bundler 1.4.0.rc.1](https://github.com/bundler/bundler/commit/cbda098d6d718f80bbcbe7fc367fbc1578be6d11) it was missing some important  features that we wanted to include. While we could have released 1.4 immediately and started testing 1.5, the 1.4 RC had significant bugs as well. While discussing our options, it became clear that the best way to improve Bundler for all users was to drop the 1.4 release and cut a new 1.5.0 release candidate that included all the features in 1.4.0 and the features we had intended to release in 1.4 originally but missed in the RC. Even though there isn't a 1.4.0, we're excited to bring an even faster Bundler to the community with parallel installs. Please help us test 1.5.0! We've compiled a list of changes in the [what's new section](http://bundler.io/v1.5/whats_new.html) on [Bundler documentation website](http://bundler.io).

Special thanks to [Jacob Kaplan Moss](http://twitter.com/jacobian) and [Daniel Farina](http://twitter.com/danfarina) for advising me on this course of action and bringing their sane Python practices with them.
