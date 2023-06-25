---
layout: post
title:  "I build apps with a unix philosophy"
tags: philosophy
author: camerondavison
comments: true
---

# Unix Philosophy

“This is the Unix philosophy: Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface.”
— Douglas McIlroy

## Piping
## POSIX arguments 

## XSV as an example 
https://github.com/BurntSushi/xsv

- - should it require the file to be named .csv to be comma delim? Or can I choose it through a command line art? 
- - sample and flatten work together.

## CRUD Application as an example

I work for [steadily](www.steadily.com) and we work with [documents](https://en.wikipedia.org/wiki/Document-oriented_database) we call policies. 

Sometimes we need to rewrite a policy, this just means that we want to copy all the data from the old policy and then start a new one.

We could implement our application many ways, but two that come to mind are:
1. Create a new endpoint that copies the data from one document and creates another one. This endpoint would also be responsible for adding and changing any data that would need to be part of the new document.
2. Use the existing Read endpoint, and the existing Create Endpoint, assuming that whoever is createing the new document will be responsible for adding the data that is helpful to know why a document changed.

There are pros and cons to both of these implementations. I would say for #1 this is more likely to always have the correct data on it, because it is being controlled by the system. 
It is not what I would consider part of the unix philosophy though because it is not working together with any other applications.
The #2 implementation I think leads it self to more being able to chain complexity. Every time we wanted to add more data during rewrite we would need to add another parameter to the new rewrite endoint. For #2 we could just add it.
Testing even becomes easier because you don't have more surface area for code. By not adding a new endpoint, the create and read endpoints continue to be the only two code paths. If read works, and create works the rewrite will work.
