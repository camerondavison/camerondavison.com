---
layout: post
title:  "Building μtrakr"
tags: utrakr,intro
author: camerondavison
---

Introduction: My name is Cameron Davison and this is a series on 
writing the https://utrakr.app url shortener.

## Introduction
I thought that it would be a good idea to start a series on writing a production quality app from beginning to end.
I am an engineer at [Digital Pharmacist](https://www.digitalpharmacist.com/) and I am passionate about writing production
software system using the [unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy) "Make each program do one thing well" 
that you can compose into a larger software system to build a business. This philosophy also tends to lead to a service oriented
architecture and solving lots of distributed system problems. I like "distributed systems" and think that running all the
software at a company is the largest distributed system there is. This puts me into leadership positions deploying code,
solving infrastructure problems, and building platforms.

## What is this series?
Often I have had the opportunity to interview new employees at my job. I like to ask more senior level engineering
candidates to design an url shortner. This is a fun question to ask because it can go so many ways depending on
what you are interested in. I thought that it would be fun to write a series of blog posts explaining how I would start
a business around url shortening.

## Goals
1. Pick a somewhat short catchy domain
1. Host a single page html app to submit long urls
1. Host a backend that saves and redirects urls
1. Log every interaction
1. Allow social logins
1. Graph and show statistics
1. Allow paying for vanity (bring your own) domains

## Plan
I am planning to cover all of these parts as I work on them and build them out.

## Pick a catchy domain
I picked μtrakr as the catchy name. The "μ" meaning micro, and trakr is short for tracker.
Usually these services call them short urls, but to be different I started calling them
micro urls. Non ascii characters are hard to find and type, so that is how I landed with
the domain utrakr.app . It is small, and .app requires https which is good. These days with
[letsencrypt](https://letsencrypt.org) you are best off trying to force everyone to use
https and set [HSTS headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)
to doubly make sure they are always using ssl. The "app" TLD requires HSTS and SSL 
so you have even less to worry about.

## Buy the domain
Once you find a domain that is available there are dozens of places that you can buy the domain from.
Buying a domain can be from a place that you feel most comfortable with or just the cheapest. You are
always able to change the [name server](https://www.cloudflare.com/learning/dns/dns-server-types/#authoritative-nameserver)
of the domain to a more reliable name server at any time. I have been using [domains.google](https://domains.google/) 
partly because it is a fun url, and because it has a really easy to use UI. It is very straight forward and
solves one problem, domain registration, well.

## Setup Email
You have the choice to set up email through a domain by pointing the [MX records](https://www.cloudflare.com/learning/dns/dns-records/dns-mx-record/)
to point to any mail server that you want. Google offers [gsuite](https://gsuite.google.com/pricing.html), which is
very popular, but it cost 6 dollars a month. 
I was looking for something low cost while started this project, but it is still good to have an email
so that you can sign up for any tools that you need to use and keep it separate from your personal email.
This is when I stumbled across
[improvmx](https://improvmx.com). ImprovMX was super easy to set up email forwarding to my personal email, while still allowing
me to have dedicated email addresses for everything related to utrakr.

## Next Steps
In my next post I would like to cover setting up a GCP account and Terraform to configure our domain and its MX records.
