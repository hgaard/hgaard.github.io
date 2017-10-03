+++
author = "Jakob Højgaard"
categories = ["blogging", "ghost"]
date = 0001-01-01T00:00:00Z
description = ""
draft = true
slug = "notes-on-upgrading-ghost"
tags = ["blogging", "ghost"]
title = "Running a Ghost blog on Heroku"

+++

#Choosing a blog
bla bla.. ghost
its simple, it's...
Used [this](http://www.therightcode.net/deploy-ghost-to-heroku-for-free/) guide by [Greg Bergé](http://www.therightcode.net/).
#Running on Heroku
Having installed my blog on Heroku and using Postgres for storing data the official upgrade [guide](http://support.ghost.org/how-to-upgrade/) is not completely sufficient. This is because Ghost by is configured to use SQLite. So I just had to remember to do a npm install.

    npm install pg --save

That checked in and pushed to Heroku and the blog was upgraded to the latest Ghost version.
