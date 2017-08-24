---
layout: post
title:  "Why Is Persistence So Hard?"
date:   2017-08-24 20:00:00 -0600
categories: design java persistence
comments: true
---
Quite a lot of Java applications, especially in the enterprise software development, have to deal with some kind of persistence. Persistence is a hot topic of lots of discussions, and many people in the industry are struggling to make it right. I won't try to provide an ideal solution to the problem, but rather try to answer the question: why persistence is it still so difficult?

### Ways of implementing persistence
The most common approach is to use some SQL database and some [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping) framework. ORM frameworks allow not to write SQL queries manually and [claim](http://hibernate.org/orm/) to provide a transparent way to work with plain objects, without worrying too much about database tables. In this case domain objects themselves don't do anything with the database, everything is done by the ORM framework.

What are the pitfalls of this approach? Well, some people [argue](http://www.yegor256.com/2014/12/01/orm-offensive-anti-pattern.html), that ORM is inherently evil, because it turns objects into plain data structures, while in a rich domain model they should implement the business logic. I personally think, that's not ORM's fault but rather developers, who lack discipline or design skills and don't prevent such a design degradation. Although, there is some truth in that. ORM is designed in a way, which forces you to open your objects internals (yeah, with getters or reflection, but still...), violating encapsulation.

A polar alternative to ORM is putting SQL related functionality directly into the domain objects. In this case SQL queries are spread around all the domain classes. What's the disadvantages of that? In my opinion, the main issue here is that SQL isn't something directly related to the domain model (unless the domain is a database itself), so it shouldn't be in the domain classes. Also, having SQL in the domain model makes testing much more difficult.

There are also some solutions in between of these two alternatives, like [Spring JdbcTemplate](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html) or [MyBatis](http://www.mybatis.org/mybatis-3/), which are supposed to make database usage easier, than with plain JDBC, but not so cumbersome as with Hibernate.

So, there are, basically, two opposite approaches to the persistence:

1) Put persistence logic into domain model objects
2) Put persistence logic somewhere else, provide a way for persistence layer to read internals of domain objects

With the option 1 we have to deal with some non-busyness logic related functionality inside the domain model, with the option two we violate encapsulation. Both options look bad to me, they are both a bit ugly in one way or another.


### Persistence in software is artificial
What is a domain model? It's basically a model of some real world domain. It includes objects, which represent some real entities. These objects have methods, which correspond to the real world logic. Objects related to each other, collaborate with each other, react to events and so on.

While a domain model can contain some more technical parts, which don't have direct analogies in the domain itself, domain model represents something real, some small part of our universe.   

What happens to physical objects?

E.g., you take a sheet of paper and a pen, you write something on the paper... It stays there, it doesn't disappear, right? You can go to a desert island without any electricity and take this paper with you... Its existence doesn't depend on any external resource of energy.  

Persistence of a real object is embedded into our reality.

What about the software? Everything inside a computer exist in the computer memory. You can't create an object outside of it. And we have this unfortunate devision of memory: it can be either energy dependent or not. We're modeling real world objects, but we can't just create them, we have to first create them in RAM (just because it's fast and convenient) and then save to some persistent storage (because otherwise we can loose them).

Programming languages support in-memory objects manipulation quite nicely. You can write code in a way, that kinda tells a story about your objects. You can make it elegant and understandable... But then you have to add persistence. And it looks like you can't get it done not ugly.


### Conclusion
I think, it's a fundamental problem of persistence. In the ideal programming world we'd be able to create objects in similarly lightweight code constructions as we do now and have transparent persistence. Just like it happens with objects in the real world.
