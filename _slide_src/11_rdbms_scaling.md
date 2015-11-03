# RDBMS Scaling and Service Oriented Architectures
.fx: title

__CS290B__

Dr. Bryce Boe

November 3, 2015

---

# Today's Agenda

* TODO
* RDBMS

---

# TODO

* Write Tsung xml file containing a single-complete session
* Use XML file to establish a baseline load test

---

# Motivation

You have your web application running on EC2. It is becoming increasingly
popular and performance is degrading.

> What do you do?

---

# Handling Concurrent Requests

You have deployed application servers that can serve many concurrent requests.

But as your site's popularity continues to grow that is not sufficient.

> What do you do?

![Concurrent Requests](img/server_process_pool.png)

---

# Vertical Scaling

You have increased your instance sizes and handled more load.

However, as the popularity continues to grow you are no longer able to continue
scaling vertically.

> What do you do?

![Vertical Scaling](img/vertical_scaling.png)

---

# Horizontal Scaling

You have introduced a load balancer that distributes traffic across a pool of
application servers.

Nevertheless, as the traffic continues to increase, additional horizontal
scaling of the application servers does not solve the problem.

> What do you do?

![Horizontal Scaling](img/load_balanced_topology.png)

---

# Caching

You have properly configured HTTP caching such that unnecessary requests never
happen.

You are also caching heavyweight database operations.

But it is still not enough to handle the increased load of your popular site.

> What do you do?

---

# SQL Structure and Query Optimization

You have reduced the number of queries your application servers makes to the
database.

You have used `EXPLAIN` to discover indexes to add, and to ensure your most
common SQL queries are as optimal as possible.

However, your database is still a bottleneck.

> What do you do?

---

# Database Vertical Scaling

> Is your database swapping (out of memory)?

Scale up the amount of memory.

> Is your database load too high?

Scale up the CPU.

> Are neither of those working?
> What do you do?

---

# Database Horizontal Scaling?

With application servers we are able to scale horizontally by adding additional
machines and introducing a load balancer to access them.

> Can we do the same with the database?

![Horizontal Database Idea](img/horizontal_database_idea.png)

---

# Non-trivial DB Horizontal Scaing

 > Can we horizontally scale the database by adding more database servers and
 > accessing them through a load balancer?

Unfortunately, it's not that simple.

Horizontal scaling works great for _stateless_ services. However, the database
contains the state of our application, thus is not trivial to horizontally
scale.

---

# Problem: R(X), W(X), R(X)

![Database Horizontal Scaling Problem](img/database_horizontal_scaling_problem.png)

---

# Scaling Relational Databases

In a future lecture we will discuss scaling the data layer using non-relational
databases (NoSQL).

Today we will discuss in what ways we __can__ horiztonally scale a relational
database.

---

# Three Techniques

Each of the following techniques partitions RDBMS in a different way:

* Sharding
* Service Oriented Architectures (SOA)
* Separating Reads from Writes

---

# AKF Cube

![AKF Cube](img/akf_cube.png)

Reference: [http://akfpartners.com/techblog/2008/05/08/splitting-applications-or-services-for-scale/](http://akfpartners.com/techblog/2008/05/08/splitting-applications-or-services-for-scale/)

---

# Sharding: Idea

Take a single database and split (__shard__) it up into multiple smaller
databases such that everything still works.

> How do we handle joins across sharded data?

---

# Sharding: Joins

Any particular database join connects a small part of your database. However,
trasitively, database joins could connect everything together.

## E.g. Demo App

* Any comment is only related to its parent (if not top-level), its children
  (replies), and its submission.
* Submissions relate to each other through communities.
* Transitively, all of these relationships could be joined across.

---

# Sharding: Separating Data

.fx: img-left

![Sharding](img/sharding_app_server.png)

Sharding a database requires finding some partition of your data that ideally
produces unrelated (not joined across) _shards_.

Once separated (_sharded_), your application cannot utilize the database to
join across them.

If you need to perform operations across sharded data, you will need to do it
at the application level. Consider the performance tradeoffs. Could you shard
another way?

---

# Sharding: Similar Data

Sharding involves splitting data of the same time (e.g., the rows of the
tables).

For instance if we wanted to shard our `Comments` table into two partitions, we
could store comments belonging to half the submissions in _shard1_, and
those belonging to the other half in _shard2_.

> What is not sharding?

Separating tables into their own databases is not sharding. While this approach
may work for some applications, the ability to join across tables is lost.

---

# Finding the Data

Assume we have sharded the data for our application. We need some sort of
mapping to determine where to find that data.

> How can we find what shard our data is on?

---

# Finding the Shard

## At the application server layer?

> How would we implement this?

## At the load balancer?

> How would we implement this?

## Across multiple load balancers?

> How would we implement this?

---

# At the App Server

.fx: img-left

![App Server Sharding](img/sharding_app_server.png)

Each application server contains a configuration that informs it of where each
database is (IP address, DNS name) and how to map data to the database.

The mapping can be arbitrarily complex.

The mapping itself may even be stored in a database.

---

# At the load balancer

The load balancer could be configured to route requests to the app servers that
are configured to _talk_ to the right database.

Such mappings are limited by knowledge that the database can inspect:

* Resource URI
* Headers
* Request Payload

![Load Balancer Sharding](img/sharding_load_balancer.png)

---

# Across Load Balancers

.fx: img-left

![Sharding Across Load Balancers](img/sharding_region.png)

Hostnames (DNS) can be configured to point to the correct load balancer for a
given request.

Examples:

* en.wikipedia.org v. es.wikipedia.org (language based sharding)
* google.com v. google.co.uk (location based sharding)
* na6.salesforce.com v. naX.salesforce.com (customer based sharding)

__Note__: The above examples could involve only a single load balancer.

---

# Finding Data Trade-offs

The approaches we just described are vary from provding more flexibility to
providing more scalability.

* App Server (most flexible)
* Load Balancer
* DNS (most scalable)

---

# Sharding and Growth

Ideally the number of your partitions increase as the usage of your application
increases.

Example:

If each customer's data can be partitioned from the others, then doubling the
number of customers doubles the number of shards.

---

# Real World Example

## Gmail

The data that represents my email conceptually requires no relation to the data
representing other people's email.

When a request arrives to gmail, they apply some mapping function to determine
which database the user's data is located on.

Should Google need to take down a database, they can relocate the sharded data
to another database, and update the mapping.

---

# Demo App Example

* Users can create and view communities.
* Users can create submissions in these communities.
* Each submission has a tree of comments.

> How can we shard this application?

---

# Sharding Demo App

## By User?

This would have difficulty as logged in users will want to see communities,
submissions, and comments made by other logged in users.

## By Submission?

This would make generating submission lists for a single community difficult.

## By Community?

Obtaining a community list may be tricky, but cleanly partitioning comments and
submissions by their community is mostly clean.

---

# Community Sharding

> How could we make community based sharding work?

We can use information in the url with any of the three load balancing
approaches.

* http://ucsb.demo.com (community subdomain)
* http://demo.com/ucsb (community path)

Either the application server connects to the right database for the `ucsb`
community, or DNS/loadbalancer directs the request to an application server
that always talks to the `ucsb` containing database.

---

# Community Sharding Success

.fx: img-left

![Sharding Submission View](img/sharding_submission_view.png)
![Sharding New Submission View](img/sharding_new_submission_view.png)
![Sharding New Comment View](img/sharding_new_comment_view.png)

---

# Community Sharding Difficulty

.fx: img-left

![Sharding Global Submission View](img/sharding_index_view.png)

* The global list of submissions.
* List of submissions by user.
* List of comments by user.

> What can we do to resolve these issues?

---

# Solving Sharding Problems

* Modify the user interface such that the difficult to shard page does not
  exist.
    * Can you get by with only providing the list of communities?
* Alternatively periodically run an expensive background job keep a
  semi-up-to-date global submission list aggregating results from accross
  databases.

---

# Sharding in Rails

Rails does not have built-in support for sharding. However, thanks to the
opensource community, there exists a great Rails sharding gem Octopus:

[https://github.com/tchandy/octopus](https://github.com/tchandy/octopus)

    !ruby
    class BrazilController < ActionController::Base
      around_filter :select_shard

      def select_shard(&block)
        Octopus.using(:brazil, &block)
      end
    end

The above sends all actions to the Brazil controller to the `:brazil` shard.

---

# Sharding Trade-offs

## Strengths

* If you genuinely have zero relations across shard, this scaling path is very
  powerful.
* Sharding works best when partitions grow with usage.

## Weaknesses

* Sharding can inhibit feature development. That is your application may be
  perfectly shardable today, but future features may change that.
* Not easy to retroactively add sharding to an existing application.
* Transactions across shards do not exist.
* Consistent DB snapshots accross shards do not exist.

---

# Service Oriented Architecture