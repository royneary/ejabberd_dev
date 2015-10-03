# My GSoC 2015 project: push for XMPP (1)

The Google summer of code is over and this is my attempt to explain what my
project was about, what I implemented and what is still to be done.
This article is meant to be an introduction to the push topic for people who
know some basics about XMPP. If you know that it's based on a federated client-server architecture
and heard about [pubsub](http://xmpp.org/extensions/xep-0060.html) and [stream management](http://xmpp.org/extensions/xep-0198.html), you will probably be fine.
This is the first part of my summary. I'll elaborate about technical details
in another article. In this part there are some references to technical
documents though for those who want to dive deeper.

## The problem
Most mobile operating systems provide a framework app developers have
to use in order to let their apps communicate with the outside. That's an
attempt to save battery by limiting CPU cycles and wireless
communication to a minimum. I'll pick iOS as an example but all operating systems I looked at follow very
similar approaches. 
I'll simplify the situation a little in order to make it
clear. An iOS app only runs as long as you look at it. As soon as you switch to
another app or turn off the screen the app will be stopped. The OS gives the
app the possibility to perform last wish operations though. Once it's stopped the app
can no longer send data or react to incoming data. If the app doesn't close
connections (and it should before it is stopped) those connections are dead
and it can take the other side a long time to realize that. The only way for
an outside sender to wake up the app and make it responsive again is to send a push
notification. [Push](https://en.wikipedia.org/wiki/Push_technology) is a term used by many protocols but when mobile OS
vendors say push they mean a very specific thing: using their proprietary
cloud service to send notification packets to users. The mobile OS listens on exactly one
connection and will wake up an app when a notification arrives for it.
As I said I simplified the situation. There are exceptions to this strict
policy. There is [background execution on iOS](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html). But in general XMPP apps are not
supposed to use that. An app which does not follow this rule might not be
allowed into the app store.

[XMPP core](http://xmpp.org/rfcs/rfc6120.html) requires a permanent TCP connection between client and server. If
a client closes the connection (or the server realizes the connection is dead)
the client is considered offline and the server won't route packets to it
anymore. So mobile XMPP apps cannot receive messages most of the time.
This is where XEP-0357 comes into play.

## The solution: XEP-0357
[XEP-0357](http://xmpp.org/extensions/xep-0357.html) is a standard extension written by [Lance Stout](https://github.com/legastero/) which defines a way for XMPP
servers to act as a push provider, that is request push notifications at the vendor's
push service if the receiving mobile device has no connection open.

The XEP divides the task of sending push packets into two parts: The *XMPP
server* part and the *app server* part.
The XMPP server's task is to manage subscriptions of push clients: they can
enable/disable push and set some parameters. Requesting push notifications at
the vendor's push service is done by another piece of software which may even run on another
machine. It's called the app server. If an XMPP server receives an XMPP
packet for a client that previously enabled push and that client does
not have an open TCP connection, the XMPP server sends all the information
needed for crafting a push notification to the app server which then crafts
and sends a push notification request to the vendor's push service.
To be exact the XEP only specifies the XMPP server part and does not tell what an app server should do (as my project is
all about push it's not hard to guess what my app server implementation does:
crafting and dispatching push notifications, yay!).
This indirect approach allows different XMPP servers to use the same app
server. Most vendors (that is all except
for Ubuntu and Mozilla) even require such a setup. When a push provider wants
to send a push notification the vendor's service asks for authentication credentials. Those
credentials are issued by the vendor to the app developer only. Thus all push notifications for a
specific app have to be requested by a service the app-developer runs.

One interesting detail which may sound a little complicated at first is *how*
an XMPP server tells an app server to send a push notification: it's done via
Pubsub. There needs to be a pubsub service somewhere where the app server can
create nodes and subscribe to them. The XMPP server then publishes the information
needed for requesting push notifications at those nodes. It sounds complicated to introduce this
extra indirection but on the other hand pubsub is already implemented and
(on paper) provides all the features needed for this task. You'll see in the
next chapter that with my ejabberd module it's possible to run all needed
components within one ejabberd instance without much effort.

Let's look at an example. Our user Alice has an iPhone with an XMPP app called *chatninja*
installed. These are the necessary steps in order to use push:

1. chatninja registers at the app server run by the developers of chatninja.
If Alice had an Ubuntu phone instead she could run her own app server or use
that of a friend. In the
registration request a token identifying Alice's iPhone is included. The app
server sends back the Jabber ID of a pubsub service, the name of an allocated node
on that service and a password.

2. Chatninja sends an enable request to Alice's XMPP server including all the
information received from the app server (Pubsub JID, node name, password).
Optionally it can configure what information shall be included in push
notifications. As chatninja is a privacy-aware app, it tells the server to
only include the number of new subscription requests and the number of new
messages but not the the senders' Jabber IDs and the message contents.

3. Alice turns off the screen. Chatninja disconnects from the XMPP server and
is stopped by the operating system.

4. Bob wants to say hello to Alice and sends her a message.

5. Alice's XMPP server cannot route Bob's message directly to her iPhone but
instead publishes an item at the pubsub service it was told about.
It needs the node name and the password it received earlier for that.

6. The chatninja app server gets notified about the published item. It crafts
a push notification request and sends it to [Apple's push notification service](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html),
including the token identifying Alice's iPhone.

7. Alices iPhone receives the push notification. Chatninja wakes up and connects
to the XMPP server to pick up Bob's message.

## Implementation
In order to understand why I did certain things it's nice to know my motivation for
working on this project. I wanted to provide a missing puzzle piece for
seamlessly working instant messaging clients on the different mobile operating
systems. As mentioned earlier XEP-0357 only specifies what XMPP server's should do and
leaves the app server part to app developers. Lance Stout told me the reason
for this: XMPP can be used for many things, not only instant messaging and
thus app developers should be able to decide when their apps receive push notifications and when not.
That's not the best foundation for the instant messaging use case because
every developer of a messaging app would have to take
care of the app server too. That's why besides the XMPP server part I implemented a generic instant
messaging app server as part of mod_push, my ejabberd module. This allows an
app user to use any server running ejabberd with mod_push installed. But
that's not all. Other XMPP servers like prosody should get invited to the
party too. So I started oshiya. Oshiya is an [XMPP component](http://xmpp.org/extensions/xep-0114.html) which is an app server
with the same XMPP interface as mod_push. So clients can choose between
ejabberd with mod_push installed or any XMPP server with oshiya connected.
Both setups should be compatible. I hope this will help spread push
in the XMPP world.

Both mod_push and oshiya will support the same set of push services (for
implementation status see the next section). These are:

* [APNS (Apple push notification service)](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html)
* [GCM (Google cloud messaging)](https://developers.google.com/cloud-messaging)
* [Mozilla SimplePush](https://wiki.mozilla.org/WebAPI/SimplePush)
* [Ubuntu Push](https://developer.ubuntu.com/en/start/platform/guides/push-notifications-client-guide)
* [WNS (Windows notification service)](https://msdn.microsoft.com/en-us//library/windows/apps/hh913756.aspx)

As I said earlier Mozilla SimplePush and Ubuntu Push are different in one
aspect. They don't require per-app authentication. This allows running your
own app server whereas on the other platforms apps have to use an app server
run by the app developer. My app server implementations allow both setups.

An important implementation detail is how ejabberd distinguishes between
connected and disconnected clients. mod_push uses stream management's
resumption feature for that. Clients are expected to close their connection
when they're about to be stopped. If a client enabled push before mod_push
will initiate push notifications from now on until the client resumes its XMPP
stream. Another approach would be to close the XMPP stream completely when the
app is stopped. The advantage of my approach is that clients will keep their
presence state which is important for MUC rooms for example.

You can find the source code and documentation of both [mod_push](https://github.com/royneary/mod_push) and
[oshiya](https://github.com/royneary/oshiya) on github. 

## Up next
Both mod_push and oshiya still require some work.
Right now mod_push is the more mature project. It made some modifications of ejabberd necessary which
are not yet merged back into master. So running mod_push currently requires my ejabberd branch.
Hopefully this will change soon.

oshiya does not support all push services yet (Mozilla SimplePush and WNS are
missing). It does not support service discovery yet either. I will try to build
oshiya packages for popular linux distributions soon (help is very welcome)
and I'll write more documentation too.

Last but not least I'll cooperate with app developers and try and help fixing
issues that might come up. I know of some people who are already actively working on
mobile XMPP apps supporting push. Off course everyone is ivited to try my
software and file bug reports.

If you wonder how mobile XMPP's future will look like you should also have a
look at [another important project](http://conversationsgsoc2015.blogspot.de/) in this year's GSoC. I'm quite optimistic.
