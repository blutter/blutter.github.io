---
layout: post
title: Octopus Deploy Timeouts and Time Travel
---

I'm working on a product that uses Octopus Deploy to deploy and upgrade sites in a production environment.
One of the recent features that I implemented is to upgrade multiple websites from our back office website. The
feature is driven by a 'dev ops' user and is meant to be relatively user friendly - to the extent that all you
do is specify the software release and version to upgrade the site to.

This story was relatively straight forward to implement - apart from a few moving parts to synchronise. The testers also
had to get all the moving parts in place to test this story.

However, in our final verification cycle we found that deployments would timeout after they are queued by the
Octopus server. On closer inspection, all deployments are queued successfully for deployment and proceed sequentially. But suspiciously
the deployments that run 20 minutes after being queued would timeout.

It turns out our version of Octopus Deploy (2.5) has a 20 minute [timeout](http://help.octopusdeploy.com/discussions/questions/5255-options-for-setting-the-deployment-queue-timeout-value)
on deployments. Newer versions (3.0+) have removed this timeout.

My organisation doesn't upgrade production tools at the drop of a hat (the joys of developing medical software).
So as a workaround I chose to periodically schedule each deployment to run in the future. This works around the 20 minute timeout.

The Octopus client API conveniently has a [QueueTime parameter](http://www.nudoq.org/#!/Packages/Octopus.Client/Octopus.Client/TaskResource/P/QueueTime).


The first set of deployments that I scheduled were configured to run immediately by setting QueueTime to now:

~~~
  QueueTime = DateTimeOffset.UtcNow;
~~~

But Octopus would occasionally complain about time travel by throwing this exception:

~~~
  Octopus.Client.Exceptions.OctopusValidationException: There was a problem with your request.

  Octopus Deploy is capable of many things, but not time travel. Please schedule your deployment to begin in the future.
~~~

Okay - so clocks [drift](https://blog.codinghorror.com/keeping-time-on-the-pc/). The clock in our back office
website which requests the deployment to run 'right now' might be ahead of the clock on the Octopus server.

The simple fix is to set the QueueTime to null as the [documentation](https://github.com/OctopusDeploy/OctopusDeploy-Api/wiki/Deployments) indicates.

But it makes me wonder how the API could be improved to reflect the realities of the real world. As much as I like
[time travel](http://www.dailymail.co.uk/sciencetech/article-3470022/Is-universe-like-flip-book-Physicists-say-new-theory-seconds-pass-help-make-time-travel-reality.html)
it's currently beyond my ability (apart from being able to travel forward in time at the rate of one second per second).

A [TimeSpan](https://msdn.microsoft.com/en-us/library/system.timespan(v=vs.110).aspx?f=255&mspperror=-2147217396#Anchor_6)
is probably more appropriate. Just queue the deployment to occur 15 minutes into the future instead of using an absolute
point in time (like Now + 15 minutes).

It also makes me wonder how I could have picked up this issue earlier. I tested with deployments which took up to about
15 minutes to commence but hitting the magical 20 minute timeout just never happened.
