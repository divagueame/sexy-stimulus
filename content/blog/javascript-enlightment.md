---
external: false
draft: true
title: Designing data intensive applications
description: .
date: 2023-11-11
---

Chapter 1 Reliability, scalabity and maintainability

Most applications are data-intensive, not compute-intensive.

It provides an interesting example of load. It presents the case of a Twitter user posting a new tweet. There might be two main approaches,
to insert new tweets into a tweets table and when a user wants to retrieve his feed, then filter out by the user's followees. The other suggested approach would be to
keep track of each users feeds when a new tweet is created, this resulting in a much cheaper operation when any user wants to read his feed, as it would have been
already computed ahead. However, writing becomes with this approach rather expensive, specially for those users who have millions of followers, as each one of those
feeds need to be updated when a new post is created. Eventually Twitter switched to a hybrid solution where the users with most followers user the first approach.

Scaling stateless services across several machines is fairly easy, but when handling data the complexity raises very fast. Therefore, databases should be scaled vertically first,
until the cost or availability requirements force to a horizontal scaling. Even though the scalabe architecture of a application is catered to its needs, they are all usually
built from the general purpose building-blocks and patterns.
In regards to maintainability, we should always keep in mind operability, simplicity and evolvability,

- Chapter 2: Data models and query languages

"Most applications are built by layering one data model onto another". At the application level, there might be APIs to manipulate real-life based data models. To
store that data, it is expressed as a general purpose format such as JSON or a table in database. Then that representation is encoded as bytes either in memory, disk
or over the network. Furthermore, those bytes are represented in terms of electrical pulses.  
Relational model vs Document model.

- Chapter 3 Storage and retrieval
