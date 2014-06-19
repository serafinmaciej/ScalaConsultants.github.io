---
comments: true
date: 2014-02-07 14:40:00
layout: post
slug: photo-service
title: Web applications with photo service - easy peasy or a hard nut
summary: Have you ever thought about creating dedicated photo service for your application? If so, then you have many possible use cases to consider. I'll try to give you a few tips based on a real life expierience, that could help in designing and maintaning such a service.
author: Arkadiusz Kaczyński
tags:
- Web app
- Photo
---

Have you ever thought about creating dedicated photo service for your application? If so, then you have many possible use cases to consider. I'll try to give you a few tips based on a real life expierience, that could help in designing and maintaning such a service.

## Overview
Even if you don't have an application with dedicated photo service, you probably thought about such solution sometime in the past, or even right now. You probably have at least a few ideas how it could look, or what tools you would choose to acomplish this task.

### Check format before resizing
There is a lot of different image formats, a ew palletes and lot of metadata hidden inside. Unfortunatelly not all standard Java/Scala libraries are able to cover it. Even if you use additional library you should implement your own check to assure, you pass only those images you can work with, otherwise you can easily end up with corrupted objects, or tidy amount of exceptions. Yet it is important to say, there is probably no 100% accurate solution here.

#### Solution
For our task we can use [Apache Commons Imaging][1] formerly known as Apache Sanselan. It is a pure java library which allows photo manipulation and format checks ([Lift][2] uses it inside its [Lift Imaging][3] module). For more details about supported formats visit project [page][4]

Simple usage for format validation:

~~~ scala
def isValidFormat(bytes: Array[Byte]): Boolean = {
    Imaging.guessFormat(bytes) match {
        case (ImageFormat.IMAGE_FORMAT_BMP || 
                ImageFormat.IMAGE_FORMAT_PNG || 
                other formats supported by your ecosystem) => true
        case _ => false 
    }
}
~~~

Imaging.guessFormat can also take a File or a ByteSource, so you should find it usable in your case as well.

As it is said on the Commons Imaging project page, it is not the fastest solution, but probably the most painless to use - you just attach project jar in your favourite way, prepare function like shown above, and you are ready with your checker prototype.

### Optimize resizing flow
This is probably the trickiest part of the whole photo service. It requires a lot of resources like cpu and memory, especially if many resizings are done simultaneously - process can easily put down large servers if is not correctly managed.

#### Solution
Firstly put your resizing code into actor (like AkkaActor in Scala). 
Probably you would like to have responsive application - choose 2 sizes that covers most of your need for given task, and prepare them "in place". Probably those sizes will be from 300px to 900px - the most universal one. 

While you can start using your photos, pass rest of the sizes to the actor. Don't do heavy operations outside an actor - this is the least sensible way of starting with it.

### Offload resizing to the dedicated machine
While we already made the first step in optimizing resizing flow, it's not the end, it's the absolute least. Still you are using your main machine for resizing - this strictly means, that your whole application, api and other crucial parts are vulnerable to short spikes, longer freezes, long loading and access time, or in the worst scenario for a downtime.
Resizing is a specific use case, which can put your server down very often, especially when more users are uploading their whole albums. You are not able to secure your availability, unless you move your resizing process to separate machine.

#### Solution
If you wish to cover it yourself, you'll need to have some shared storage for passing input and output (resized) files and a bridge which will take care of communication. With more specific needs this would be the best idea, but when you can live with 3rd party service here, I would suggest using one of the cloud service such as [cdnconnect][5] or [cloudinary][6]. Both services can transform as well as deliver images through delivery network, this means you can use it as your resizing tool or as facade of your photo service.


### Unsatisfactory photo quality
It's likely you will notice, while using some standard Java librariers, that quality of transformation is not good enough for your needs - this is quite common problem. If you start with BufferedImage and ImageIO you will notice that the quality of output image is not always sufficient. You can try using render hints or other mechanisms available for those classes, but I would suggest something else according to "do not reinvent the wheel" motto.

#### Solution 
Use already available tools - take a look at [imgscalr][7], the java image scalling library based on Java2D and addressing all the common problems you can face while trying to get things done with writing your own code. It takes care of resizing your images to desired size, while maintaning aspect ratio.

The simpliest way, which will resize image up to 200px, while keeping aspect ratio (image is an BufferedImage type input):

~~~ java
BufferedImage resized = Scalr.resize(image, 200);
~~~

This is the really fast startup of working with images in java, while push away a lot of problems with tweaking settings and writing unnecessary code. If you want to have more control over resizing process you should use a Scalr.Mode and Scalr.Method.

### Design your resolutions well
Your application can be a desktop page, mobile page or mobile apps on different platforms. When it comes to photo service they do not differ much, at least not from the general point of view. Problem is when your desktop application grows, your company evolves, and you introduce mobile apps one by one.

Imagine you need few different sizes on desktop page, 2 or 3 more on mobile page, and a few more for mobile apps solutions. There can be even worse scenario when your mobile apps designs will require different sizes for each platform. 

You can end with ca. 10-15 different image sizes (+ probably the original one) to fullfill design wishlist. In other words your resizing process will run 10-15 times per uploaded photo! It can put down even powerful server during single photo upload, while I assume you will allow to upload multiple photos (whole albums) simultaneously. Also it is mentioning that you can exceed single model object size quite quickly. Also when you think about professional photo service you will surely store uploaded file as well.

#### Solution
In this case designs and backend must collaborate tightly to work out the best solution. Try to keep down the resolutions number by replacing similar sizes with only one - bandwidth usage extension will be negligible, but you save amount of resizing cycles, used cpu and memory. 
There can also be situations where storage is your limitation like in MongoDB where a maximum size of a document is 16MB. Actually in MongoDB you have pretty nice alternative to use - GridFS, which is more appropriate for such cases.

Also think about what photo types you will need and where - maybe your design will allow to have optional fields or more photo type oriented solution.

For most cases 30px, 60px and 120px should cover avatar requirements, while additional 2-3 sizes (300-400px and 700-900px) should cover main features needs. You will probably also need 2 sizes with higher resolutions for providing excellent user expierience.


### Cropping images and impact of new resolutions
When you are working with album photos, you rather won't need such options, but when users are allowed to upload their avatars, then they are going to miss cropping very soon. 

It can look like an easy and straightforward part, but believe me, you can set it up in a very wrong way. How will you cropp images? You will surely have some preview instead of working on full sized image. Let's look how it can be done.

#### Solution
Simplicity is the goal here. It doesn't matter much what you use on client side, the main rule here is to pass to the backend function coordinates embedded in the original image resolution. 
If your solution is based on a preview, you will have to rescale them before sending image to the sever. Backend shouldn't deal with such rescalling on it's own, it's only task in this matter is to cropp and save avatars by working on a uploaded image, with coordinates based on it.
Such coordinates can be saved to database and used in the future in case you will introduce a change in your design.

If you proceed in a other way, you will surely cause a lot of problems, like loosing an ability to recrop photo - don't do this!
  
### Performance & stability issues
You will not be the first who face a performance issue while working with photos. Likely you will observe resource exhaustion, service temporary unavailability, long access times or even more than a few downtimes. Dangling exceptions are also a part of this game.

#### Here are a few tips in such case:
- offload resizing to dedicated machines and threads. The ideal solution is when your application server is used only for serving your web page content/api.
- do not wait hoping for luck. When you notice something that goes wrong - act immediately (look into logs, profile your application with a tool like [visualvm][8], review your code for potential holes, stress testing the application). Problems such as these do not repair themselves neither they happen once. Ask your team for code review and help in stress tests because doing it yourself will be long, paintfull and probably fruitless.
- to be sure you are not missing anything prepare email template and send it when something goes wrong
- 3xTest! - test, test and test your service whenever you can, with different sources (different colour palletes, different formats, and metadata included; use photos with different colours, content, sizes, and aspect ratios)
- improve your format checker whenever you can - better checker means more secure solution
- when you looking into the logs, search for exceptions first
- when working with exceptions think about good place of catching them; do not catch and throw next exception instead  - this is a really bad habit. If you use Scala, use Option - remember, it is always better to deal with such situations in nice manner with you in the control, than being a passenger.

### Summary
As you probably realized by now creating a photo services is not an easy matter, however if follow a few simple rules, you will surely be successfull.


[1]: http://commons.apache.org/proper/commons-imaging/
[2]: http://liftweb.net/
[3]: https://github.com/liftmodules/imaging
[4]: http://commons.apache.org/proper/commons-imaging/formatsupport.html
[5]: http://www.cdnconnect.com/
[6]: http://cloudinary.com/
[7]: http://www.thebuzzmedia.com/software/imgscalr-java-image-scaling-library/
[8]: http://visualvm.java.net/



