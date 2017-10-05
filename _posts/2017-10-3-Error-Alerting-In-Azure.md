---
layout: post
title: Error alerting in Azure
---

So, I have a service running in Azure and at one point I decided it is high time to start logging errors I have and to get somehow notified about them.

Azure has some astonishing number of things inside that can be integrated with each other. Often there are many ways to do the same task using different Azure components and those solutions eventually differ mostly in price and flexibility. 

In this post, I am going to describe one way of reaching the goal from the title and to point pros and cons of the resulting solution.

## 1. Start logging to Azure

Usually in .NET for logging purposes one uses some third-part library like [NLog](http://nlog-project.org/), [log4net](https://logging.apache.org/log4net/) or [SeriLog](https://serilog.net/) which all have simple APIs and may differ in the way to set up the logger. For F#, a special library exists called [Logary](https://github.com/logary/logary). 

But with Azure things can be even simpler, one does not have to install any third-party stuff to get it working. Two things need to be done.

### 1.1 Turn on logging in the Azure app

Here it is: _Monitoring_ -> _Diagnostics Logs_ -> _Application logging (blob)_:
![Azure Logging Settings]({{ site.baseurl }}/images/post-1/azure-logging-settings.png "Azure Logging Settings")

I turned it on and set Storage Settings. Right from this window it is possible to create a Blob Storage where all the errors will go.

### 1.2 Use Trace class methods from System.Diagnostics

Now, when I need to log some error I just open `System.Diagnostics` namespace and call `Trace.TraceError` method.

After running this in Azure I saw my blob storage filling up with messages. They are put in different folders depending on current time. Path format is _/year/month/day/hour/_:
![Errors blob storage]({{ site.baseurl }}/images/post-1/errors-blob-storage.png "Errors blob storage")

A log file is a csv file. There one can see date and time of the message, message itself and a few other things.
![Blob properties]({{ site.baseurl }}/images/post-1/blob-properties.png "Blob properties")

Depending on [privacy settings](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-manage-access-to-resources) for the storage, it can be possible to access the blob by link or only to download it from this window. 

Another good thing about `Trace` class methods is that when the app is running locally the messages are seen in the _Output_ window.

## 2. Alert errors with Logic App

[Logic Apps](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-what-are-logic-apps) are yet another Azure service to manage some collateral routines. Once I was impressed by [this video](https://www.youtube.com/watch?v=faIvOYpcUq4) where the guy easily sends emails from Azure and even creates Wunderlist tasks which I am a big fan of. I decided to give Logic Apps a try.

### So close...

Logic Apps are about integrating so at first I looked if there already exists some trigger about Blob Storage. I searched for "Blob Storage" and... Yes it exists!
![Blob storage trigger]({{ site.baseurl }}/images/post-1/blob-storage-trigger.png "Blob storage trigger")

Seems to be just the right thing. But here is the problem. One needs to specify the exact container path there:
![Blob storage trigger: specify container]({{ site.baseurl }}/images/post-1/blob-storage-trigger-specify-container.png "Blob storage trigger: specify container")

It is not possible in this case because with time blobs keep appearing in new folders. I was desperate to find some wildcards but all in vain. So, I had to go for less elegant solution. 

One way was to manually control where log blobs go. That meant I needed to  fall back to logging libraries, [create](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-dotnet-how-to-use-blobs) some _CloudBlobClient_ in code and make it send all the logs in one folder which then could be accessed by the trigger specified above. 

This was feasible, however I did not like the idea of having all the logs in one folder and wished to avoid writing code for cross-cutting concerns in my app.

Another approach is to go hard with Azure Functions that can be [integrated](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob) with blob storage as well. [Here](http://www.chrisjohnson.io/2016/04/24/parsing-azure-blob-storage-logs-using-azure-functions/) a guy goes for it when having a similar problem. But I felt it is also too cumbersome and just too much coding for such a task. 

I decided to stick with Logic Apps and see how it goes. Here is my way.

### 2.1 Manually schedule the task

For this, I used the _Recurrence_ trigger:
![Schedule the task]({{ site.baseurl }}/images/post-1/schedule-the-task.png "Schedule the task")

I set it to run at the end of every hour.

### 2.2 Write Azure Function to format blob path

It should be a dumb function returning current date and time in specific format. There are many templates for Azure Functions and not all of them are pluggable to Logic Apps. The simplest working thing is _HttpTrigger_. I chose F# for it and called it _GetPathToBlobs_.

I removed all the boilerplate and here is the code:
![Azure Function code]({{ site.baseurl }}/images/post-1/azure-function-code.png "Azure function code")

A few things to consider:
* Parameter should stay in the function, otherwise it fails at compile-time or runtime.
* Two-digit formatting is used for months as Azure puts September logs in the folder _09_ not _9_.
* Capital _HH_ formatting is used for hours as Azure puts 2 p.m. logs in the folder _14_ not _2_ or _02_.

### 2.3 Connect Azure Function to the Logic App

I searched for Azure Functions actions and selected the function written in the previous step:  
![Connecting function to logic app]({{ site.baseurl }}/images/post-1/connecting-function-to-logic-app.png "Connecting function to logic app")

No advanced parameters are needed, no input is to be specified.

### 2.4 Get blobs with logs

For this, I used _List blobs_ action:
![List blobs]({{ site.baseurl }}/images/post-1/list-blobs.png "List blobs")

Logic apps allow to use output from previous steps as input for the current one. Here, I need _Body_ from _GetPathToBlobs_ function response.

### 2.5 Get logs from blobs

The next action is _Get blob content using path_:
![Get logs from blobs]({{ site.baseurl }}/images/post-1/get-logs-from-blobs.png "Get logs from blobs")

For blob path I specified _Path_ from the previous step. After that my Logic App automagically created Foreach loop to repeat the routine for every blob in the list. 

At this point, Logic App looked like this:
![Foreach created]({{ site.baseurl }}/images/post-1/foreach-created.png "Foreach created")

### 2.6 Send logs by email

The only thing left is to actually alert somewhere about what has happened. 

Logic apps have nice connectors with email services. I chose _Gmail - Send Email_ action specifying my email address and passing _File Content_ from the previous step to the body of the message.

Eventually, the whole logic app looks like this:
![Whole app]({{ site.baseurl }}/images/post-1/whole-app.png "Whole app")

If something bad happens, at the end of that hour I receive a message like this:
![Log alert]({{ site.baseurl }}/images/post-1/log-alert.png "Log alert")

Formatting is not pretty but the main thing is message says what happened and when.

## Conclusions

So, I would note the following pros of my solution:
* The Logic App seems clear and, well, logical.
* There is some space for configuration, e.g. the task can be fired more often, the letter can be nicer formatted and so on.
* Little code is required, only one small function with 6 lines of code. No need to track or to maintain it. No cross-cutting concerns code in the app.

Some drawbacks:
* 5 steps in the logic app for such a task can still seem to be an overkill.
* It is a timer-driven app instead of event-driven. If there is nothing new in the blob storage, failed requests (blobs not found) mess up Logic App activity log. 
* Responsibility of checking if the blob is new would lie on Logic App's designer.

Although at the first glance it may seem that Azure services can do everything for you, they may be not as flexible as wanted. There is still some space for inner services' integration improvement. 

Also, I do not really feel comfortable with the idea _"there is more than one way to do it"_ which drives Azure services, but this is obviously a matter of taste.
