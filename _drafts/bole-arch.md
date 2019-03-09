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
     
### Overall problems to solve

1. Logs off the machine

    The first problem to solve is how to reliably get the logs off of the machine.
    We want to get the logs off the machine because in the current modern datacenter it is easier to think of machines as cattle instead of pets and assume they can disapear at any time.
    If the machine is gone, and it had the logs then you will never be able to figure out why. Also, we want to get the logs off of the machine quickly because, nobody wants disk alerts going off because of logs, so logs are also removed quickly.
1. No missing logs

    There may be a way to sample the logs in the future but I am currently working under the assumption that we do not want to lose any of the logs. Losing logs will likey happen in times of machine stress, which is usually when you want to know "why" it is stressed.
    This means that we should mark the logs with some kind of identifier and make sure that they all make it off of the machine correctly.
    
1. Dedupe logs
    
    Most likely during the attempt to get everything off of a machine we will replay some portion of the logs so we are probably going to want a way to dedupe them. This can be hard because sometimes the logs will be exaclty the same. It may be logging a request at multiple come in on the same second. So we will likely need to encode something else besides the message into the log in order to correctly dedupe.
    
1. Correct timing

    If we do not parse the logs (or they do not even log a timestamp) we need to make sure that the timestamp is as close to the time that the log was "emitted" instead of the time that it was "ingested". The time can lag behind if bufffered for a while before timestamp metadata is added to the event.
    
1. Replication

    Once we get the logs off of the machines we want to make sure that they are replciated so that we do not still run into a problem with one machine taking out some of our logs.
    
1. Queriable

    The logs need to be stored in such a way that we can query them. Without being able to easily find the specific log we are looking for the logs will be useless.
    [ELK](https://www.elastic.co/elk-stack) allows you to use distirubted lucene queries to accomplish this, but requires that the logs be parse already before fed into the elasticsearch cluster. [Loki](https://grafana.com/loki) which was inspired by [oklog](https://github.com/oklog/oklog) takes more of a distributed grep with some tagging and mostly timestamp indexing. I think that there is an opportunity for something inbetween that keeps the raw logs, providing grep and re parsing of the raw logs but with a way to parse the logs on the fly into an inverted index for quicker future lookups if the parsing stays the same.

### Bole Architecture

#### Collector

This runs on every server. It could be a sidecar if you want, but likely needs to just be configured once for each machine.
It can be configured for each type of logs that needs to be collected. 
* files
    
    Even with docker you can set `--max-size` on the [json-file](https://docs.docker.com/config/containers/logging/json-file/) logger to get it to roll, and will log somehere like `/var/lib/docker/containers/<container id>/<container id>-json.log` in linux.
    
* journald
    
    These logs are stored in a binary format that can be parsed.
