---
layout: projects
title: "Realtime Web Part 1: Admin Architecture Experiment"
---

Many webapps I've built for work and on the side have been [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) applications with little to no business logic. At first, I manually built them with Flask. Hand-writing repetitive HTML templates, forms, and queries was tedious. Then I discovered [Flask-Admin](https://flask-admin.readthedocs.io/en/latest/), which provided a 100X productivity boost. Flask-Admin generates a web site for the administrator to access and modify the database tables. This is useful for administrators and it also speeds up development of table-centric business applications. In addition, admin frameworks can easily be extended when the use-case calls for something other than a table (images, graphs, dashboards, etc).

Screenshot of a basic Flask-Admin interface: 

![](/projects/assets/realtime_web_part_1_admin_architecture_experiment_1.jpg)

One drawback of Flask-Admin is that the configuration of its frontend has to be coded in the backend. This means that business users have to iterate with the developer on simple formatting changes. Another drawback is that Flask-Admin does not provide real-time updates. The web page must be refreshed to see new, updated data. This may seem like a trivial nuisance in the abstract but it's incredibly irritating in practice. Contrast it with the user experience of collaborative editing on a Google Doc.

Frontend requirements:

1.  Single page application
2.  Generates routes for each database table and view
3.  Displays data in HTML tables that are easy to sort and filter
4.  Updates displayed data without a refresh based on incoming websocket messages
5.  Allows the user to configure what columns are displayed and how they are formatted

Options for the frontend

1.  [React](https://facebook.github.io/react/)'s [react-admin](https://github.com/marmelab/react-admin)
2.  [Angular](https://angularjs.org/)'s [ng-admin](https://github.com/marmelab/ng-admin)
3.  Write the frontend from scratch with React or Angular

Backend requirements:

1.  Reflects the database schema's tables, views, and relations
2.  Uses the schema to generate authenticated REST endpoints
3.  Knows what data the client currently has so it knows what updates to send
4.  Sends updates from PostgreSQL's LISTEN/NOTIFY interface via an authenticated websocket session

Options for REST (POST/GET/PUT/DELETE)

1.  Use [Flask-RESTful](http://flask-restful.readthedocs.io/en/latest/), [Django REST framework](http://www.django-rest-framework.org/), or the equivalent to build an API
2.  Query a thin API around the database like [PostgREST](http://postgrest.com/) or [PostGraphQL](https://github.com/calebmer/postgraphql)
3.  Write a custom REST framework from scratch with [Express](https://expressjs.com/) or [aiohttp](http://aiohttp.readthedocs.io/en/stable/)
4.  Connect to the database directly

Options for real-time websocket updates

1.  Use an already-built real-time web framework like [Mojolicious](http://mojolicious.org/) or [Feathers](http://docs.feathersjs.com/)
2.  Get [Meteor](https://www.meteor.com/) to work with PostgreSQL
3.  Use [Flask-SocketIO](https://flask-socketio.readthedocs.io/en/latest/) or [Django Channels](https://channels.readthedocs.io/en/latest/)
4.  Write a custom websocket server
5.  Connect to the database directly

This table by [Brent Tubbs](https://bitbucket.org/btubbs/todopy-pg) illustrates how I envision the frontend and backend communicating: 

![](/projects/assets/realtime_web_part_1_admin_architecture_experiment_2.jpg)

Since I'm most experienced with Flask and Python I'm going to start by experimenting with Flask-SocketIO. With that baseline in mind I'll try Mojolicious and Feathers.
