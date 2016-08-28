---
layout: post
title: LinkThrow is Now Live!
---

I often find myself reading some long web page on my Mac, and wishing I had an easy way to
"throw" it over to my iPhone so I could continue reading on the go. Emailing the link to 
myself was one way to handle it - and with the Messages app, I could "text" the link to myself - but I always wished there was an easier way.

Apple's Handoff functionality is great, but you have to be using Safari - and, well, I prefer Google Chrome.

So when I saw the linkthrow.com domain was available, I snatched it up!  And then...sat on it for three years without doing anything.  But I finally set aside a couple weekends to work on it, and the site is now live: [LinkThrow.com](http://www.linkthrow.com)

It only works with Chrome and iOS right now, but I'll probably build an Android app and some more browser extensions if there's any demand.

### How it works

It's pretty straightforward:

* Install the extension for Chrome.

* Install the mobile app on your iPhone or iPad - be sure to choose Yes when asked if you want to receive push notifications.  It won't work otherwise.

* The mobile app will give you a six-character code - enter that code in the browser extension and you can start throwing links.  Throwing a link to the phone sends a push notification, and sliding to open the push notification launches the page in mobile Safari.

### What I learned

This was also a learning exercise for me.  I hadn't had much opportunity to play around with MongoDB, so I'm using that for the back end.  Spring Data made it very simple to integrate with the app.  I did find Mongo's security a bit confusing - I somehow ended locking myself out of my own database!  I finally decided to leave "auth" disabled and just limit access using Amazon's VPC permissions.  Easy enough to just tunnel in if I need to.

I also had never done a Chrome extension before, so that was a first.  It's basically just a block of HTML with some Javascript to handle the Ajax calls.  What's nice is once I add my iPhone on one Mac, I can throw to it from all my Macs, since extensions are tied to your Google account and shared across all devices.