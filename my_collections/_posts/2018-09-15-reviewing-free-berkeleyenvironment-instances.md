---
layout: post
title:  "Reviewing 'Free BerkeleyEnvironment Instances'"
---

[In my last pull request]({{ site.baseurl }}{% link _posts/2018-09-04-fixing-berkeley-environment-strpath.md %}) I 
started my solution in the `GetWalletEnv` function which manages access to `BerkeleyEnvironment` objects. I noticed a 
comment stating `the map will be changed in the future to hold pointers instead of objects`. Sure enough, there was a 
10 month old pull request to do just that, [#11911](https://github.com/bitcoin/bitcoin/pull/11911). I reviewed the code 
and I added a few 
[comments](https://github.com/PierreRochard/bitcoin/commit/ea7f8291dab051cceb2751ef40f7d2a0ee67f95c) 
and [unit tests](https://github.com/PierreRochard/bitcoin/commit/fc47a8843b03718529c402fd42cabb7a0834a9ff).
