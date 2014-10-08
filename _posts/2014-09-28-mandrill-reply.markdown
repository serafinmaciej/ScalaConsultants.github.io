---
comments: true
date: 2014-01-13 14:10:22
layout: post
slug: about-add
title: About API driven development
summary: Quick tutorial how to get setup a mail reply with Play 2.x and Mandrill.
author: Patryk Jażdżewski
tags:

---

Hi everyone. Today I'm going to show you how to setup a mail reply feature using Play 2.x backend server and Mandrill email service. On the internet you can find informations about distinct features, but not how to glue it all together. This post will guide you step by step to create a working solution. Originally I created the reply while working for an awesome startup called JobHive, so it might get a bit specific, but using the general idea and code example you can prepare something on your own.

### Why?

As you probably know many web apps, forum and social media sites use email as a way of notifying their users about activity on site. Portals like Github and others go even further. As most notifications are one-way only, Github allows users to communicate via email in an biderctional manner, you can not only receive emails but also reply to them. Which in some cases might be super useful. 

I was tasked with doing such a thing for our conversation system. When somebody sends you a message and you aren't logged in the message also lands in your email inbox. Since you can receive an message via email it seems logical to allow replying via the same channel.

We don't want to deal with technicalities of mail servers so we delegated this work to Mandrill.

### How?

The general plan is as follows:

* Setup the mail server to allow replies
* Send a message in such a way that an answer is possible
* Process the reply

## Setup Mandrill

## Creates inbound routes in backend server

## Send prepared messages

## Parse responesma

### Links

[Inbound Email Overview](http://help.mandrill.com/entries/21699367-Inbound-Email-Processing-Overview)

[Response format](http://help.mandrill.com/entries/22092308-What-is-the-format-of-inbound-email-webhooks-)
