---
layout: post
title:  "Cookies and JavaScript"
date:   2013-07-25 08:17:36
category: javascript
tags:
- cookies
- javascript
- pairing
---

Yesterday I spent part of my day pairing with a development team that is transitioning from a .NET system to Rails, so they have a unique opportunity to simultaneously refine and create from scratch. It was really awesome to be part of a team that knew where they were, had been, and wanted to go.

I was working with one of their devs and we were tasked with presenting users visiting the site using IE9 or lower a gentle alert that would prompt them to use a more updated browser. The catch was that we only wanted to show this to the user once, we didn't want them to see it every time they logged in. The portion of the site we were working with used Bootstrap, so we already knew that we wanted to use a [modal](http://twitter.github.io/bootstrap/javascript.html#modals) to display the information to the user.

Getting the window to display only for IE9 and lower was very easy due to the built-in support IE has for some conditional syntax. We set up a jQuery function to show the modal if that conditional triggered. Easy enough!

The next step took some searching, but it was surprisingly easier than I thought it would be. We knew that we wanted to store something in a cookie so that the user wouldn't have to see the alert again. We considered using Rails cookies, but it felt like an unnecessary and clunky extra step. We were already using JavaScript to display the modal, so why not use that same function to create or read a cookie also?

Neither of us had manipulated or accessed cookies using JavaScript before, so we took to Google to find out how to do it. We stumbled upon this awesome library of functions for [working with cookies in JavaScript](http://www.quirksmode.org/js/cookies.html) and created a permanent cookie storing function:

{% highlight javascript %}
function createPermanentCookie(name, value) {
  var expires = "; expires=Fri, 31 Dec 9999 23:59:59 GMT";
  document.cookie = name + "=" + value + expires + "; path=/";
}
{% endhighlight %}

And borrowed the existing read cookie function:

{% highlight javascript %}
function readCookie(name) {
  var nameEQ = name + "=";
  var ca = document.cookie.split(';');
  for(var i=0;i < ca.length;i++) {
    var c = ca[i];
    while (c.charAt(0)==' ') c = c.substring(1,c.length);
    if (c.indexOf(nameEQ) == 0) return c.substring(nameEQ.length,c.length);
  }
  return null;
}
{% endhighlight %}

Easy enough! Actually, a lot easier than I expected it to be.

The best part? I wrote code that now lives in their application and is being used everyday by grandparents the world over who refuse to stop using Internet Explorer.