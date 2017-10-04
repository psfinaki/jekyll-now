---
layout: post
title: Error alerting in Azure
---

So you have a service running in Azure and at one point you decide it is high time to start logging errors you have and to get somehow notified about them.

Azure has some astonishing amount of things inside that often could be integrated with each other. I suppose there are many ways to do the same task using different Azure components and those solutions eventually differ mostly in price and flexibility. So, in this post, I am going to describe one way of reaching the goal from the title and to specify its advantages and disadvantages.

## 1. Start logging to Azure

Usually in .NET for logging purposes one uses some third-part library like NLog, log4net or SeriLog which all have simple APIs and may differ in the way to set up the logger. For F#, a special library exists called [Logary](https://github.com/logary/logary). 

But with Azure things can be even simpler, you do not have to install any third-party stuff to get it working. Two things need to be done.

### 1.1 Turn on logging in the Azure app

Go to the app, then Monitoring -> Diagnosting Logs -> Application logging (blob):
![Azure Logging Settings]({{ site.baseurl }}/images/post-1/azure-logging-settings.png "Azure Logging Settings")

Turn it on and set Storage Settings. Right from this window it is possible to create a Blob Storage where all the errors will go.

### 1.2 Use Trace class methods from System.Diagnostics

Right, whenever you need to log some error you just open/use `System.Diagnostics` and call `Trace.TraceError` method.

After running this in Azure you will see your blob storage filling up with messages. They are put in different folders in the following fashion:
![Errors Blob Storage]({{ site.baseurl }}/images/post-1/errors-blob-storage.png "Errors Blob Storage")

A log file is a csv file:
![Blob Properties]({{ site.baseurl }}/images/post-1/blob-properties.png "Blob Properties")

Depending on [privacy settings](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-manage-access-to-resources) in the storage, you can be able to access the blob by link or only to download it from this window. In that file one can see date and time of the message, message itself and a few other things.

Another good things about `Trace` class methods is that when the app is running locally the messages are seen in the Output window. 