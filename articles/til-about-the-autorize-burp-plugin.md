---
description: >-
  Autorize is a great plugin for Burp that lets you quickly and easily test for
  Authentication (AuthN) and Authorization (AuthZ) issues within webapps.
---

# TIL about the Autorize Burp Plugin

_Originally Published:  February 10, 2016_

#### Auth issues are hard <a id="auth-issues-are-hard"></a>

AuthN and AuthZ issues can be tricky and time consuming to track down. You have to take a potentially vulnerable request and run it through a gauntlet of tests \(unauthenticated and different user levels\) which typically involves juggling several cookies per request. Most engagments are on a time limit so being thorough here is an investment, but when it pays off it's totally worth it.

#### Enter Autorize <a id="enter-autorize"></a>

Autorize allows you to easily take a cookie from one request and swap it into a second request. There are a handful of plugins for Burp that handle this but what sets Autorize apart is how easy it makes things. Everything is driven by the right click menu on a request. You right click the request you want the cookies from, send them to Autorize and then right click the request you want to test and send that over. From there it attempts three requests.

* A baseline request, using the original cookies
* A test for AuthZ issues, by swapping in the cookies that you specified
* An unauthenticated test, by clearing out the cookies

What's particularly helpful is that you can toggle it to run for every request as you browse the site. This way, as you're browsing the site you're automatically testing those requests using another user account as well as unauthenticated. Results are displayed in an easy to parse table.

Very cool stuff.

You can install Autorize from the Bapp store. Their code is available on [github](https://github.com/Quitten/Autorize).

