---
layout: post
title:  "Designing Distributed Logging - Bole"
tags: design bole
author: camerondavison
---

This is my first time trying to write up a design documentation, but there has to be a 
first time for everything.
I am hoping that it will be a good way to get my thoughts organized.

### Bole
What is in a name? For a while now I started naming things abstractly so that they 
can morph as need be into what they need to become without the distrations of someone
trying to understand if the name fits or not. 
Bole is "the trunk of a tree" so it seemed like a good abstract name for something that deals with logs.
Recently I read about [withoutboats](https://github.com/withoutboats) 
[having a similar](https://boats.gitlab.io/blog/post/names-and-scuba/) if not that same philisophy.

### History
I have read/used over the years a variety of logs. I think that it can be good
to talk about the history of logging to get a idea what the goals should be. 
Trying to go in order from simplest to most complex.

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
     - requires you to setup something like 
     [filebeat](https://github.com/elastic/beats/tree/master/filebeat) to ship them off somewhere
     
### Overall problems to solve

1. Logs off the machine

    The first problem to solve is how to reliably get the logs off of the machine.
    We want to get the logs off the machine because in the current modern datacenter it 
    is easier to think of machines as cattle instead of pets and assume they can disapear at any time.
    If the machine is gone, and it had the logs then you will never be able to figure out why. 
    Also, we want to get the logs off of the machine quickly because, nobody wants disk
    alerts going off because of logs, so logs are also removed quickly.
    
1. No missing logs

    There may be a way to sample the logs in the future but I am currently working under
    the assumption that we do not want to lose any of the logs. Losing logs will likey 
    happen in times of machine stress, which is usually when you want to know "why" it is stressed.
    This means that we should mark the logs with some kind of identifier and make sure 
    that they all make it off of the machine correctly.
    
1. Dedupe logs
    
    Most likely during the attempt to get everything off of a machine we will replay 
    some portion of the logs so we are probably going to want a way to dedupe them. 
    This can be hard because sometimes the logs will be exaclty the same. It may be 
    logging a request at multiple come in on the same second. So we will likely need 
    to encode something else besides the message into the log in order to correctly dedupe.
    
1. Correct timing

    If we do not parse the logs (or they do not even log a timestamp) we need to make 
    sure that the timestamp is as close to the time that the log was "emitted" instead 
    of the time that it was "ingested". The time can lag behind if bufffered for a 
    while before timestamp metadata is added to the event.
    
1. Replication

    Once we get the logs off of the machines we want to make sure that they are replciated 
    so that we do not still run into a problem with one machine taking out some of our logs.
    
1. Queriable

    The logs need to be stored in such a way that we can query them. Without being able to 
    easily find the specific log we are looking for the logs will be useless.
    [ELK](https://www.elastic.co/elk-stack) allows you to use distirubted lucene queries 
    to accomplish this, but requires that the logs be parse already before fed into the 
    elasticsearch cluster. [Loki](https://grafana.com/loki) which was inspired 
    by [oklog](https://github.com/oklog/oklog)
    takes more of a distributed grep with some tagging and mostly timestamp indexing. 
    I think that there is an opportunity for something inbetween that keeps the raw logs, 
    providing grep and re parsing of the raw logs but with a way to parse the logs on the 
    fly into an inverted index for quicker future lookups if the parsing stays the same.

### Bole Architecture

#### Bark (outermost part of the trunk)

This runs on every server. It could be a sidecar if you want, but likely needs to just 
be configured once for each machine.
It can be configured for each type of logs that needs to be collected. 

* files
    
    Even with docker you can set `--max-size` on the 
    [json-file](https://docs.docker.com/config/containers/logging/json-file/) logger to get it 
    to roll, and will log somehere like `/var/lib/docker/containers/<container id>/<container id>-json.log`
    in linux.
    
* journald
    
    These logs are stored in a binary format that can be parsed.
    
Since this process is running on each machine, it can keep its state local. Its job 
is just to help with getting the logs off of that machine as quick as possible, and 
augmenting the logs with the unique identifier (for deduplication). Part of the state
will be tracking line number in files that have already been ingested, and which id
what assigned to which line. Also it will need to keep track of file renames. We
do not want to send the same events again for a file rename, which will likey happen
in the event of a log roll. We do not want to have to wait for the roll to happen
since this will put arbitrary lag on the events that come in.

The local `bark` should assign a unique ID to a given line in a given file.
Only after the ID has been created and persited will the line be transmitted off of
the local machine. This way we can transmit with retries but only have 1 ID per line,
even across restarts of the machine. This same technique can be used for the journald
system, just using the [`__CURSOR`](https://www.freedesktop.org/software/systemd/man/systemd.journal-fields.html#__CURSOR=).
Bole uses [monotonic ulid](https://github.com/ulid/spec#monotonicity) since they are unique, 
easy to pass around in url params, and also lexigraphically sortable. All great properties
for log entries. As long as we assign these ID's as quickly as possible locally, even
before transmition, we will more likely have the correct timestamp for the logs. We
could optimize something in the future to read the timestamps from journald or configure
where the timestamp is in the log if that does not give us the correct timestamps.

We desire to get the logs off of the machine as quickly as possible, but also the machine
is a very convinient place to keep the logs for buffering. In the event that we cannot
push the logs anywhere we keep assigning IDs but but keep track of where we are in transmitting 
seperately. Also we should keep track of another cursor which will be acknowldegement from the
`ring`s that the logs of been persisted and replicated. Ideally the `bark` will not
delete any logs until it gets this `ack` (but we may not be in controll of that). Once we
get an `ack` from the `ring`s we are allowed to stop tracking the log line to ID tuple, but we do 
want to keep track of the fact that the line was fully processed. On restart always send
every line that is older than the latest `ack` again, which should be okay since the event will be
deduplicated based on the ID.

#### Ring

The first task of the `ring` is to get events from the `bark`s bundle them up into
`segment`s. These should be time base and size based, they should be compressed (since 
logs usually compress really well), and then deduplicated.
The output is a bunch of `segment`s that will be replicated, and consumed by searches.

In order to do replication the `ring` nodes will need to keep a memberlist of all the
currently known nodes. This can be kept through gossip, maybe something similar to
[memberlist](https://github.com/hashicorp/memberlist).

Queries are run against every node in the system, the originator of the query knows all
of the other nodes in the `ring` since it is itself part of the gossip pool. 
We are deliberatly sending to every node so that we do not have to coordinate who
is the owner of the node. This costs a little more but is unexpected
to be a bottle neck. The `ring` nodes should be considerably less than the `bark` nodes.
The results of the queries are then k-way merged and deduplicated by the client that originated
the request.
