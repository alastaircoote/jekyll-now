---
layout: post
title: UNNotificationServiceExtension and memory
---

As previously noted, my hacky attempt at making a service worker implementation really came undone when it tried to handle push messages. UNNotificationServiceExtension has a hard memory limit, and if your code hits that limit it'll immediately terminate the process without letting you change the content of a notification. Not great.

So I've been trying to find a way around this. Testing the limit is very difficult, because you can't use the service extension with a local notification - it has to be a remote one. So I've upgraded my phone to the iOS 11 beta (more fool me) and created a tiny test app that sends HTTP calls to a service that'll send me a remote notification. Then, it checks to see if the UNNotificationServiceExtension successfully manipulated the content of the notification or not. *Then*, in the extension itself, I'm spinning up a JSContext, and executing code inside it that creates a `Uint8Array` of ever increasing size, until it reaches a point where the extension crashes.

My initial tests showed that it reached this around the 900KB mark. Not awful (at least it runs), but really not great either, as this is an entirely empty JSContext, and I still need to populate it with the various worker APIs we depend on, which will take up more and more memory. Worse, it turns out that iOS doesn't close the extension when it has processed a notification - it keeps it alive for some amount of time (I have no idea how long) to process a second notification, should it arrive. There was some kind of garbage collection issue that meant the JSContexts were not being collected, so when a second notification arrived the extension tried to make a *second* JSContext and crashed immediately.

Not good at all. But, with garbage collection on my mind (which we can't control in Swift), I tried something - recreated the extension in Objective C. And, somehow, the amount of memory it could use kept ticking up and up, reaching 10MB. Great news, but I immediately had nightmares of having to make the entire library in Objective C, which I don't know well at all. But to my surprise, pulling in my ServiceWorker Swift module into the Objective C extension worked, and without decreasing the memory limit.

I don't really understand why this works, and I'm waiting to trip up on something major, but for now I'm making the bulk of the code in Swift, and then going to Objective C to write my (pretty small) notification extension code. I already ran into one issue - if I don't unassign the exceptionHandler of the JSContext it has the same issue as the Swift version - so I'll have to be vigilant and run this test app on a regular basis to make sure I'm not making any references that lead to the JSContext sticking around. But overall, good news - it looks like we'll be able to process push messages.