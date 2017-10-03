+++
author = "Jakob Højgaard"
categories = ["RethinkDB", "NoSql", "Getting Started", "Realtime"]
date = 0001-01-01T00:00:00Z
description = ""
draft = false
image = "/images/2016/03/Rethink-Logo-1.png"
slug = "doinng-stuff-with-rethinkdb"
tags = ["RethinkDB", "NoSql", "Getting Started", "Realtime"]
title = "Doing stuff with RethinkDB"

+++

I recently started a new job. Actually I moved from Copenhagen to Brisbane, Australia with my whole family, just to try living and working in a different country - and of course to give our 3 kids and ourselves a great experience/ adventure. Anyway, I got a job in the Australian consultancy [Readify](http://readify.net) and one of the perks there is PD or Professional Development for up to 20 days a year. 20 days where I can work with/learn whatever I feel like (well almost. I need to justify the relevance of the topic and I have to produce something that my colleagues can benefit from too). Yeah I know pretty sweet! So the last 2 days I have had my first PD and my goal was to get familiar with [RethinkDB](https://www.rethinkdb.com/).

Well the above is not completely true anymore. I started out writing this post when I actually did my PD, but it turns out it takes time writing stuff. Well, months have passed but here we go:

## What is RethinkDB

From [RethinkDB](http://rethinkdb.com):

>RethinkDB is the first open-source, scalable JSON database built from the ground up for the realtime web. It inverts the traditional database architecture by exposing an exciting new access model – instead of polling for changes, the developer can tell RethinkDB to continuously push updated query results to applications in realtime. RethinkDB’s realtime push architecture dramatically reduces the time and effort necessary to build scalable realtime apps

A few things to highlight here: Yeah, it is just a document DB, but with the access model kinda turned on it's head. This means that there is no longer a need to poll the database for updates in specific tables, they will just get pushed directly to you. In fact you can subscribe to chances for any query you write and these changes will be pushed directly to your app. That is, I think, very very powerful! RethinkDB also supports things like joins, subqueries, geo queries and map-reduce. On top of this, the fact that it is built as a distributed system, just makes it all the more interesting.

**Update**

*As Henrik Andersson has pointed out in the comments, I was incorrect about what kind of queries change feeds can be applied to. It is for instance not yet possible to apply change feeds to joins. For a complete description of change feeds have a look at their description [here](https://rethinkdb.com/docs/changefeeds/ruby/)*

### Why should I care

First of all think about any time you would push stuff, either to a mobile phone or to a browser. How do you usually handle that? A pattern I see very often these days in applications, where we want to ensure that all users are up to date, is the use of event/message aggregators or internal busses, where the code that writes to the database, also triggers a message to the event aggregator/bus. For these scenarios integrating RethinkDB would render the bus/aggregator redundant. Event subscribers would  simply subscribe to a query in the database instead. 

Other examples could be any collaborative applications where users look at the same data, IoT or just plain Pub/Sub scenarios.

## What did I want to do

OK, RethinkDB is some interesting tech and I wanted to get to know it little better. I decided that a logging dashboard would serve as a good example app. A simple website displaying log messages from other systems and updating the dashboard instantly when new messages arrive, simples!


### How did I do it and how to get started.

First of all, to get started with RethinkDB I needed to have it on my machine. I was on Windows for this one, so I just downloaded the latest version [here](https://www.rethinkdb.com/docs/install/windows/) and added to my tools folder (which is already in my PATH). Staring RethinkDB on windows is just running the ```rethinkdb.exe``` file. The data will by default be located in the same folder as where RethinkDB is started from. So to use a specific path for the data you would start RethinkDB with the ```-d``` switch and provide a path.

#### The Driver

In order to connect to the database I will need a driver for it. RethinkDB officially supports drivers for JavaScript, Python, Ruby and Java, but there are community developed drivers for heaps of other [languages](https://www.rethinkdb.com/docs/install-drivers/) too. I'll be writing in C#, and luckily there is a community driver for that. Actually there are two with slightly different approach to solving the problem. I picked the one maintained by [bchavez](https://github.com/bchavez/RethinkDb.Driver). It seeks to follow the official Java driver and extends it with a more C#-ish syntax and a Linq layer. The Linq part I think is especially cool since it took a bit of time for me to internalize the default syntax. 

#### Writing queries and subscribing to feeds.

To get a feel for the database and the [ReQL](https://www.rethinkdb.com/docs/introduction-to-reql/) query language I started out writing simple queries from LinqPad (I have a few examples [here](https://github.com/hgaard/RethinkLogs/tree/master/query-samples)). RethinkDB also comes with a dashboard (running on localhost port 8080) that has a *data explorer* where you can write queries in the native ReQL language (JavaScript/JSON). The data explorer provides an intellisense-like experience, that proved really helpful for discovering the API surface.

![Image of a console example query](/content/images/2016/08/rethinkdb-console-api.png)

#### Writing the app

In order to focus on the dashboard of the app I used [Serilog](https://serilog.net/) and the Serilog [RethinkDB Sink](https://github.com/serilog/serilog-sinks-rethinkdb) to produce some "real" log entries.

The simple dashboard I wanted to build would just query a single table in the database and send that back to the browser.

```csharp
var result = R.Db(Constants.LoggingDatabase).Table(LoggingTable)
                .OrderBy(R.Desc("Timestamp"))
                .Limit(50)
                .OrderBy("Timestamp")
                .RunResult<IList<LogEvent>>(_connection);
```

This will the return latest 50 entries from the ```LoggingTable```. The generic ```RunResults``` will serialize the results back in a list of ```LogEvent```'s. I'll be using a query like that for the initial loading of the system.

In order to have updates streamed back to the app from the database I will have to subscribe to a change feed.

```csharp
    var feed = R.Db(Constants.LoggingDatabase)
                .Table(Constants.LoggingTable)
                .Changes()
                .RunChanges<LogEvent>(connection);
```

This will return a **cursor** object which in turn will return all updates for the query it represents. Think of the cursor as an IObservable collection. The generic ```RunChanges<LogEvent>``` will ensure that the results are returned as LogEvent objects. The driver also has a non generic version of the ```RunChanges()``` method which will return results as ```dynamic```. The objects that are pushed to the client is actually not only changes, but rather 2 objects, the new state and the one before that.

#### Putting it all together

So for the logging dashboard, I wanted to subscribe to all changes from the logging table, and then push these to all clients (dashboard clients). For this I used SignalR. Adding this to the change query above, it looks like this:

```csharp
    public static void HandleUpdates(IConnectionManager connectionManager)
    {
        var hub = connectionManager.GetHubContext<LogHub>();
        var connection = R.Connection()
                        .Hostname(Constants.Host)
                        .Port(Constants.Port)
                        .Connect();
        var feed = R.Db(Constants.LoggingDatabase)
                    .Table(Constants.LoggingTable)
                    .Changes()
                    .RunChanges<LogEvent>(connection);


        feed.Select(x => hub.Clients.All.onMessage(x.NewValue)).ToList();
    }
```

A final detail worth mentioning is that in order to ensure a long lived feed/cursor, I run the query as a long running task separate from the dashboard itself.

```csharp
      Task.Factory.StartNew(() =>
      {
          LogHub.HandleUpdates(connManager);
      }, TaskCreationOptions.LongRunning);
```

You can find the complete sample application with (some basic) getting started instructions on [Github](https://github.com/hgaard/rethinklogs) including simple LinqPad query examples.

## Notes on the development experience

First of all I have to say that the official documentation on RethinkDB is excellent and very elaborate. It is usually very easy to find answers and even googling for something usually takes you to their own documentation. It also has a very good description of their overall architecture and their architectural decisions. A good example of this of the description of the trade-off's they have made in regards to the [CAP theorem](https://www.rethinkdb.com/docs/architecture/#cap-theorem).

Another interesting note is the [Jepsen tests](https://aphyr.com/posts/330-jepsen-rethinkdb-2-2-3-reconfiguration) on distributes systems correctness. It seems to indicate that the RethinkDB team has a good grip on the distributed part of building databases and are very responsive when it comes to solving problems.

## Questions

I also gave a lightning talk (slides [here](https://speakerdeck.com/hgaard/introduction-to-rethinkdb)) on this topic at out last [Back2Base](https://twitter.com/mouna1619/status/733515269109243907) event at Readify. Some of my colleagues asked a few good follow up questions that I thought I might share here too. 


**What concurrency model is used?**

From the RethinkDB [blog](http://rethinkdb.com/blog/lock-free-vs-wait-free-concurrency/) there is and explanation of how RethinkDB implements a lock-free system. The tl;dr of the is; 

>Lock-free algorithms increase the overall throughput of a system by occasionally increasing the latency of a particular transaction. Most high- end database systems are based on lock-free algorithms, to varying degrees

From a [architecture documentation](http://www.rethinkdb.com/docs/architecture/#how-does-rethinkdb-execute-queries), there is a more in depth description of concurrency control.

**Would I user it production?**

The answer I gave at Back2Base was, that for something like dashboarding yes. RethinkDB seems very mature and has a fast growing community around it. For "regular" documentDB stuff, I can't really say. Mostly because I haven't used other documentDB/NoSql in important production scenarios i.e., I haven't felt the pains, and it is therefore difficult to compare the experience. But looking at the architectural decisions, the Jepsen tests and the community around RethinkDB, I would certainty not hesitate to give it a try. 


## Useful resources

* RethinkDB [documentation](https://www.rethinkdb.com/docs)
* RethinkDB [Driver](https://github.com/bchavez/RethinkDb.Driver) by Brian Chavez 
* Pluralsight - [RethinkDB fundamentals](https://www.pluralsight.com/courses/rethinkdb-fundamentals) by Rob Conery
* The [Changelog episode](https://changelog.com/181/) with Slava Akhmechet
