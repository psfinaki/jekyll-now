---
layout: post
title: Error alerting in Azure
---

So you have a service running in Azure and at one point you decide it is high time to start logging errors you have and to get somehow notified about them.

Azure has some astonishing amount of things inside that often could be integrated with each other. I suppose there are many ways to do the same task using different Azure components and those solutions eventually differ mostly in price and flexibility. So, in this post, I am going to describe one way of reaching the goal from the title and to specify its advantages and disadvantages.

## 1. Start logging to Azure

Usually in .NET for logging purposes one uses some third-part library like NLog, log4net or SeriLog which all have simple APIs and may differ in the way to set up the logger. For F#, a special library exists called [Logary](https://github.com/logary/logary). 

But with Azure things can be even simpler, you do not have to install any third-party stuff to get it working. Two things need to be done.
