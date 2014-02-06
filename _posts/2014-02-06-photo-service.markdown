---
comments: true
date: 2014-02-06 13:06:00
layout: post
slug: photo-service
title: Photo service as a part of your web application.
summary: Did you ever think about creating dedicated photo service for your application? If yes, then you have many possible use cases to consider. In this series l try to give you few tips, that could help in designing and maintaning such service.
author: Arkadiusz Kaczyński
tags:
- Web app
- Photo
---

Did you ever think about creating dedicated photo service for your application? If yes, then you have many possible use cases to consider. In this series l try to give you few tips, that could help in designing and maintaning such service. 

## Overview

Photo service can be a small part or a game player in your business. In the modern world quality of such service can affect your karma even if it's not a main part of your application - people do not like to watch blured, slow loading images, especially in social/sharing oriented applications. Now you probably asked yourself "easy to say, be more specific..." - let's try to clear the situation.

## Predesign phase

### User look - what user could expect

Before you will design strict requirements rethink your idea of this service - do not think like a programmer, architect, or product owner right now. Try to think about your service as a particular user of your application. What such user could expect from your application (based on application background and target)?. Let's think about such areas:
- uploading/delivering photos to application
- organizing them in some kind of containers (e.g albums)
- browsing through photos (how it should look like?)
- sharing and downloading (should I be able to download it, download whole album, or only share it through application, or 3rd party service?)
- platform of access (desktop site, mobile page, mobile applications)

While working with this list do not forget about application target

Write down your answers and thoughts, ask team for do the same and talk about results. You should end with short summary document, describing vague UX requirements, which will be one of the entry points for stating the features later.

### Owner look

In previous section we cared only about application influence on user perception. Now it's time to think about our look as an owner and a businessman, yet in very vague view:
- how I see my service from look and feel point of view
- should it bring "big" part of my income/increase karma (what "big" means to you; how this one corresponds to other main features - let's mark features with priorities based on value in our feeling)
- how many resources I can engage
- how big will be the load on this service (how many photos at a time span to resize/upload)

Remember, you will not be able to satisfy everybody, there will be always a percentage of people that would ask for a feature covering their use case - you must be able to make the difference between application use case consequent from background, target or a real market need and a feature change owing from one-man point of view. It's not easy, it's really hard to make it, especially when your product is entering the market.

### Result of predesign phase

As a result of predesign phase you should have a list containing overall view on your service - like main requirements, user entry points and a your plan as an owner. This will help you design your service in next step

## Design phase

### Prepare use cases
Now it is a time to use our vague requirements and prepare use cases for our application. Here you should think how user will interact with your service. Put effort in it - describe entry points, what you want to achieve and how. Be as much as specific as you can - it will be longer and more "paintfull" now, but you will have well designed view of what you want to serve.

Cooperate with team, mock things and play with simillar services - build your UX knowledge base. Last, but not least - do not put aside things, right now is the time for it.

### Technical background
When your use cases are ready, sit down and talk about strictly technical view of your service. Your dev team must know what you want to achieve and prepare your infrastructure:
- talk about types of photos (avatars, album photos, thumbnails etc) - this will have impact on other things
- talk about max resolution, count and size of input files (maybe you would like to allow to initialy resize pictures on client side to e.g spare client packets)
- describe resolution which you will use across application (and probably mobile apps) - try to narrow it, by unifing them in simillar bounds (e.g do not resize images to resolutions like 150x150, 160x160, 140x140, instead unify it to 150x150 in all designs - this will spare your storage and resizing resouces like time, cpu and memory)
- choose your storage wisely (the best way which will fit your needs and available infrastructure)
- discuss possible change of requirements in the future - your dev team will be able to prepare a more bullet-proof solution
- think about way of resizing (see next section for more details)
- do we need cropping, how we will proceed with it
- think about things like cache, compressing and way of delivering content from application to client
- try few available libraries for resizing (including language native libraries, and solutions outside your main language)

### Resizing & cropping photos - example scenario
This is probably one of the trickiest part in designing photo service. It is easy to make it in an unoptimal way, which will put even best servers down.

Let's think about such scenario:
<pre>
You have nice application that allows to upload whole albums. Each photo should have few resolutions, that you use across your service. Some of them are used as thumbnails, some of them as nice, big album photos. 
In timespan of 5 minutes 500/1k/2k/... of users upload their album photos (let say 50/100/200/500/... each one).
<p>
How your server will behave (single node, or even multinode/cloud with scalling)?
How you will proceed when you introduce new size in the future?
</p></pre>

This can be pretty scary huh? Do not worry I will give you few tips:
- do you need all the resolutions at one time? You can prepare 1-2 most universal sizes "in place" and pass rest of work to actor, while being able to serve content very fast)
- offload most work to separate thread/actor or even dedicated instance
- find optimal tools for your needs during development - play with them, do not stick with one 
- make stress tests of your service, do not trust your luck here
- prepare model in a way that will allow you to manipulate your data later(store original file, store cropping bounds according to original file, or even consider having optional resolution for different photos types) - of course this depends on your specific needs
- prepare rescue rezising actor - if something will go bad, this actor should fix your photos

## Summary & tips
In this entry I've touch only some layers of designing such services and hopefully it will be userfull for you within your process.

In the end I would like to share few tips that can spare you a lot of time:
- prepare solution in bullet-proof way - there is a lot of formats right now, and not all libraries are able to cover them. Watch out for formats with special data information, different color palette etc. Use tool for format check, and do not allow to pass format which you can not transform into rezising function
- if your rezising solutions does not give you quality you would expect, try different one e.g. see how [ImageMagick Studio][1] does the job
- try not to reinvent the wheel, especially when it comes to resizing function - try to find a ready one
- 3xTest - test you solution heavily with different files (colorful, text, greyscale, different ratio etc.), resolutions, formats and extensions
- when something goes wrong - check usage of your resouces, address logs and make more 3xTest ;)

## Usefull links:

- [ImageMagick][2]
- [ImageMagick Studio][1] - an online tool for IM
- [imgscalr][3] 


[1]: http://www.imagemagick.org/MagickStudio/
[2]: http://www.imagemagick.org/script/index.php
[3]: http://www.thebuzzmedia.com/software/imgscalr-java-image-scaling-library/



