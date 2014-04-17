This document describes issues we've run into when processing the data that diasend provides to users.

## Basal rates:

Diasend's XLS files have the "delivered" basal rates in them.  Whenever your basal rate changes, they have an event for what was delivered.  This is good compared to Medtronic's Carelink data, which leaves out some transitions when temp basals are running.  But, this has its own set of problems:

### This doesn't actually provide any indication for how long the basal should be operating. 

It is impossible to differentiate between a user that has a single basal setting and a user that went on a pump vacation.  We realized this because it happened to us.

We have a person, let's call them Eric, who just uses a flat basal rate.  As far as data from diasend is concerned, there is only one indication of this basal rate (when it started) and it just wants you to assume that the rate is continuous.  Ok, we can make that assumption.  Next thing we knew, however, Eric decided that they wanted to try out a different pump that doesn't connect with diasend.  They do that for a month, come back to their old pump and start using that.  They upload their data to diasend and we load it up.

What we see in the data is that Eric has his final basal rate from before trying out the pump and then a month later, he gets another basal rate in his data because he's back on his diasend-connected pump.  According to the previous logic, Eric's basal rate didn't change over that period of time.  However, we know that the basal rate actually didn't exist, because the pump wasn't used.  So, the assumption that the stated basal rate is in effect until it is not, is no longer valid.  What do we do?

Well, luckily(?), when you do a site change, the pump that Eric uses generates a 0 basal rate and then goes back to the scheduled rate once he is done.  So, we know that every 3~5 days, we should have a zero'ing out event to end our extended basals.  This means that in a case similar to where Eric just falls off the face of the earth for a month, we can just cap the basal rate out at a max of say 120 hours (or 5 days) after the last basal rate that we saw.

So, now instead of a 35-day-long basal, we have a 5-day-long basal.  Still an artifact, but better, I guess?  What would be really better?  Well, that would be if diasend were to provide an event every time a schedule goes into effect.  A single-rate basal schedule is effectively programmed as a rate that always begins at midnight.  Given that it always begins at midnight and is scheduled to last for 24-hours, it would be great if diasend were to guarantee that there were at least one event every midnight, even if the rate doesn't actually change.

Even on top of that, we could accidentally believe that a basal is operating for 24 hours beyond when a pump is shut of.  If diasend could also provide a "pump turned off" event or some other indication that the pump is no more, that would be extra bonus points.  Enough extra bonus points and you get free brownies, so they are highly recommended.
