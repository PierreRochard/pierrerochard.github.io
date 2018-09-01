---
layout: projects
title: "Realtime Web Part 2: Flask with Socket.io and PostgreSQL"
---

My working demo: [https://github.com/PierreRochard/realtime-flask-experiment](https://github.com/PierreRochard/realtime-flask-experiment)

While doing research I came across a project that did what I was looking for: send realtime updates from Flask to the browser. Check out Luke Yeager's demo [here](https://github.com/lukeyeager/flask-sqlalchemy-socketio-demo). The only wrinkle was that he is sending the update to a message queue at the [application level](https://github.com/PierreRochard/flask-sqlalchemy-socketio-demo/blob/master/manage.py#L31) while I'm looking to do it at the [database level](https://github.com/PierreRochard/realtime-flask-experiment/blob/master/realtime/database/create_triggers.sql#L26).

There are trade-offs between these two approaches. For performance or scalability I would have to run empirical tests. Architecturally, I think it shouldn't matter what client modifies the record: a notification should always be emitted. To make that happen without repeating notification code in every client app we have to use PostgreSQL's LISTEN/NOTIFY feature. Having said that, it may be the case that those notifications should go to an intermediate message queue like Redis or ZeroMQ. TBD.

I took Luke's demo and extended it to use Brent Tubbs [pgpubsub](https://bitbucket.org/btubbs/pgpubsub) module instead of Redis or ZeroMQ. The relevant code is [here](https://github.com/PierreRochard/realtime-flask-experiment/blob/master/realtime/database/pgpubsub_client.py). Pgpubsub is a light but incredibly useful wrapper around [raw SQL](https://www.postgresql.org/docs/current/static/sql-listen.html) queries, [select](https://docs.python.org/3.5/library/select.html), and [psycopg2](http://initd.org/psycopg/).

The minimalism of Luke's demo led me to an interesting insight. He uses DOM manipulation with Javascript to [append to an HTML list](https://github.com/PierreRochard/flask-sqlalchemy-socketio-demo/blob/master/myapp/webserver/templates/index.html#L24-L26). I extended this to update or delete existing list items based on incoming socket.io messages [here](https://github.com/PierreRochard/realtime-flask-experiment/blob/master/realtime/webserver/templates/index.html#L36-L52). I realized that I could do this same kind of DOM manipulation with a Flask-Admin Jinja2 template, even if it's rendered server-side. This deviates from my initial plan to create a REST API and use client-side rendering with React, but it would allow me to leverage what the Flask-Admin team has already built.
