---
layout: post
title:  "Designing Distributed Logging - Bole"
tags: design bole
author: camerondavison
---

This is my first time trying to write up a design documentation, but there has to be a first time for everything.
I am hoping that it will be a good way to get my thoughts organized.

### Bole
What is in a name? For a while now I started naming things abstractly so that they can morph as need be into what they need to become without the distrations of someone trying to understand if the name fits or not. 
Bole is "the trunk of a tree" so it seemed like a good abstract name for something that deals with logs.
Recently I read about [withoutboats](https://github.com/withoutboats) [having a similar](https://boats.gitlab.io/blog/post/names-and-scuba/) if not that same philisophy.

### History
I have read/used over the years a variety of logs. I think that it can be good to talk about the history of logging to get a idea what the goals should be. Trying to go in order from simplest to most complex.

- single file growing forever
  - when it gets too big what do you do?
- roll log every N bytes
  - will roll in the middle of newline
  - many logs will fill up the disks
- write to syslog locally
  - lets the OS deal with rolling and retention
- write to journald (like syslog)
  - by far my favorite (since journal out of the box has a max size and great search)
- write to a remote syslog
  - basically pushes the problem onto some other machine
  - can drop logs during high load (probably when you need them the most)
  - IMHO the same solution as SaaS offerings
- write locally with rolling but fixed number of rolled files
  - can still roll in middle of newlines (which could be fixed)
  - requires the logs be sent somewhere quickly since they can disappear in high volume
     - requires you to setup something like [filebeat](https://github.com/elastic/beats/tree/master/filebeat) to ship them off somewhere
     
