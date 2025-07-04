---
# 
---

# Don't expose your database to the world 

*Published: Fri, 4th July 2025 by Michael Ishri*

I'm so excited. I explored [NativePHP](https://nativephp.com/) when I first heard about it for the desktop but I didn't really have a need for it at the time. I had a few ideas but nothing I was sure I wanted to spend a heap of time building. But when they announced support for iOS I was intrigued, then they quickly followed with Android support. I was in!

I would consider myself a hobbyist, Laravel developer. I work for a large enterprise that is crusty and old and has no interest in modern tooling, so I'm stuck with what they dictate, so Laravel is just a thing I do out of interest. Having said that, I've worked in enterprise environments for a long time and understand a bunch about good architecture / practices.

I forget that sometimes. That not everyone has my experience. So this post is for those that are exploring NativePHP on mobile and think that they need to connect their mobile application directly to a MySQL/Postgres database. 

**Short answer, please don't!**

When you're developing a "regular" Laravel web application, your application will typically get deployed onto a web server which has a direct (and reliable) connection to your database server / service. Since your web server is considered a "secure" server, it is fine to have your database credentials somewhere your application can read from and use. Essentially it's a closed loop between your web server and database server.

Database connections are typically stateful connections, meaning you connect to the server with your credentials then a session is established without having to re-authenticate again on subsequent operations. The network protocol to talk to your database server (engine) is designed to work in this way. Stateful connections are more sensitive to packet loss and are typically "heavier" than stateless connections (like HTTP). This is because both the client and server need to maintain the sessions, instead of closing the connection after the response has been sent (like in stateless connections). So the trade-off is usually speed (stateful) vs resilience (stateless).

There is an exception to that speed argument when using a stateful connection model on unstable networks (such as mobile networks). Since there's overhead when re-establishing sessions.

Having said all that, PHP is slightly different, because every request is a new request. In PHP, a new connection to the database is established for each web request, so even though the connection remains stateful, it is short-lived...typically (we could veer off here and discuss connection pooling and persistence, but those topics are for another day). This is probably why you don't run into connection limit issues (too many connections to the database server), because PHP will release/close the connection gracefully.

When building you NativePHP mobile application, you can't guarantee a safe, secure, reliable environment (if it's being distributed publicly). So you should have a layer of abstraction in-between your application and the database server. When working with Laravel, this would usually be a REST API.

So essentially, you would build REST API endpoints which require authentication, that return JSON to mobile application which is then consumed by your application.

From a user flow perspective,
1. The user installs your application
2. The user provides their credentials to log into your application
3. After successful auth, an API token is generated and sent to the user's device and stored securely on the device
4. This API token is then used under-the-hood to interact with your API

This means you don't need to expose your database to the outside world. You continue to just expose HTTP to the outside world which is easy to secure. You can easily set up key rotation on the server side (if you want to automatically expire the API token after a period of time). It's a stateless connection so able to recover on flaky connections plus a whole bunch of other benefits.

This API token approach is just one way to handle this, there are other ways such as using JWT's, but in my opinion, when working with Laravel, this is the simplest. Check out the [Laravel Sanctum](https://laravel.com/docs/master/sanctum) docs to understand how to implement this in your application.

If we didn't take this approach, let's quickly imagine what could happen. 

You would need to figure out a way to store your database credentials on the client device. Just putting this in writing is making me scared...It's just a bad idea. We can't assume the client device is secure or will remain secure. We don't want hundreds of users making direct connections to your database server with the same credentials. If those credentials are compromised, then your database could get leaked. Since everyone would be using the same credentials, then everyone would have access to everyone else's information and because it's a direct connection to the database, it means your database would need to be exposed to the outside world. Once you have those credentials, you can just open MySQL workbench and start running arbitrary SQL queries. There are just too many issues to list with this scenario.

So then lets assume you have some way of creating a new user account on the database server for each user of your application. You would still need to expose your database server to the outside world and if your user figures out their credentials to their database user, they could then again use MySQL workbench to just run arbitrary SQL queries. Let's hope you've set up the correct access levels for your user. You would need to implement row level access which is extremely granular and again, this whole thing is just a bad idea.

The thing is, Laravel just makes building and securing API's so easy, there's little reason to not use it. I can only think of one example where you would need to store your database credentials in the client application. If you were building a database management tool like TablePlus and you wanted to have an "auto-login" feature for your user. That's it really.

So, please don't expose your database server to the outside world. Keep it behind a secure network and use a REST API instead. Have authentication on your API and only send data your application needs, nothing more (avoid sending internal ID's, timestamps, sensitive data, etc).
