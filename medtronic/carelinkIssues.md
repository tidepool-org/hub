This document is an attempt to document the issues we have run into when dealing with Medtronic CareLink records.

### `Dual/Square` bolus events can exist without a `Dual/Normal`.

This appears to happen when a patient goes through the bolus wizard, chooses to do a dual and then sets the initial injection to 0 units.  The unit appears to not emit the `Dual/Normal` because it is 0 units.  This makes the logic to properly correlate boluses with their wizard more difficult.  It would be nice if `Dual/Normal` were always emitted, no matter what.  However, this issue can also be resolved by a solution to the next item.

### Bolus records and Bolus wizard records are difficult to correlate

When a patient goes through the bolus wizard to pick a normal bolus, it generates two events

1. `BolusWizardBolusEstimate` 
2. `Bolus` with `"Bolus Type"=Normal`.

If it's a dual wave bolus, then there are three events:

1. `BolusWizardBolusEstimate` 
2. `Bolus` with `"Bolus Type"=Dual/Normal`.
3. `Bolus` with `"Bolus Type"=Dual/Square`.

These events all come from the same patient action, but we must infer the relationship based on the ordering of the `Raw-Seq Num` field across equivalent `Raw-Upload ID` values.  It would be a lot easier to manage the data if there were some key that the device attached to all events that came from the same action.  Something like a `Raw-Join Key` that you could just look for equality and know that the events are correlated.

### CareLink data doesn't come back in event order

There are two orderings to CareLink data.  The timestamp order and the `(uploadId,seq_num)` order described above when talking about correlating Bolus records above.  The `(uploadId,seq_num)` ordering is more interesting from a data correlation standpoint, but the timestamp order is more interesting from a user visualization standpoint.  CareLink delivers data in timestamp order, and there isn't a readily apparent pattern to how events with the same timestamp are ordered.

Because of the out-of-order delivery of events, we are forced to do a significant amount of sorting and re-sorting of the data in order to do various operations.  This wouldn't be an issue if there were a `Raw-Join Key` field on the data as described above.  And that would be the prefered solution for this issue.  But, short of that, it would be nice if the timestamps were such that you could order based on the tuple `(timestamp,uploadId,seq_num)` and get a well-ordered sequence of events such that correlated events happen in order.

### CareLink provides no way of separating device streams

If, for example, your pump breaks.  It's under warranty so you talk to Medtronic and get another pump, that new pump will be the same exact model as your old one (that's how these things work).  Your new pump will have a different serial number, so it is different.  

You go and upload data to CareLink, that goes fine.  Wonderful.  You now export your data to CSV and you notice that in the top of the preamble for the CSV, you have something that looks like

```
Pump:,Paradigm Revel - 723,#123456
Pump:,Paradigm Revel - 723,#654321
```

So, yay, it registers that there are two pumps with different identifiers.  You now continue on to the actual CSV data and look at the `Raw-Device Type` field.  It looks like

```
Paradigm Revel - 723
```

You go back up to your list of devices and try to figure out which one it is.  You don't know.  So, while CareLink reports are able to tell you things like "these are the device settings for unit with serial number XYZ", it is actually impossible to extract that same information from the CSV data that CareLink provides.

### Resume events do not reference their corresponding suspend

Similar to bolus and bolus wizard records, the act of suspending and resuming a pump is a set of linked events that are separate, but together form a single unit of activity.

A resume always couples with a suspend.  Without a suspend, a resume is meaningless.  Similarly a suspend without a resume is generally meaningless for the purposes of visualization (it usually indicates that a pump has stopped being used as the final suspend will happen before the data is uploaded and then data from that device will never be seen again).

Currently, CareLink simply provides an indication that the device was suspended and an indication that it was resumed.  Theoretically, this should work fine, however, when you mix it with the inability to actually distinguish one device from another, you get into a bit of a conundrum.  

What you want to do is take all of the "status" events for a given device and connect the suspends with the resumes.  However, when you change pumps from a broken pump of the same model as your new good pump and you don't handle the data uploading just perfectly, you can quite easily end up in a case where you have a suspend event from the broken pump, another suspend event from the working pump and then a resume event from the working pump.  Now you have two suspends followed by a resume and you don't necessarily know which one is correct and which one to ignore.

From looking at the Daily Summary Report in CareLink, when this happens, CareLink seems to take the heuristic that the shortest possible gap in suspension is "correct" and it ignores the other suspend event.  This appears to be the same if you end up with doubled up "resume" events.  At Tidepool, we take a slightly different tack:

1. We first group the data by its upload, if a suspend and resume both existed in the same upload, then they belong together.  We mark them and store them as such.
2. The remaining suspend/resume events (the ones that span upload boundaries), are then associated with each other on a per-device (or as close to it as we can get) basis.

When we associate events together, we assign an id to the "suspend" event and then in the "resume" event's metadata, we attach a "joinKey" which points back to the suspend event in question.  This allows downstream processing to not have to guess at which datum is effected by which event.  Medtronic should adopt the same strategy when storing the actual events from its pumps.  That is, when it goes into suspend, it should "id" the suspend event and store that.  When it comes out of suspend, it should then generate an audit log with the id from the suspend event stored in it and clear the stored id.

### Temp Percent Basals stop `BasalProfileStart` events

In the CareLink csv, there are events called `BasalProfileStart`.  These signify the start of a new basal rate "on schedule".  I.e. they happen for each change in basal rate that occurs because of your basal schedules.  They happen pretty regularly and have a nice amount of information in them.  These events were pretty well thought out and are generally useful.

Except, you notice that your BG is declining a little fast after that meal.  You were trying to decide if you wanted to bolus 4 units or 5 units and ultimately went with 5, but it looks like that was wrong.  You want to stave off a low by decreasing your basal rate.  So you go and put in a 4-hour temp basal of 20%, given your schedule, this will hold back roughly a unit of insulin, things seem to even out and you have a good day of it.  Congratulations, you diabetes ninja you.

You then go to pull this data out of CareLink and you want to see just how much that temp percent basal affected your schedule.  You grab your trusty CareLink csv file and go to town on it.  You find the `ChangeBasalTempPercent` event that lists a duration of 4 hours and start looking for the `BasalProfileStart` events that would have been modified so that you can figure out exactly how much insulin must not have been delivered.  Your schedule changes the rate of insulin every hour, so the amount of insulin must've changed as well.

Much to your dismay, the first `BasalProfileStart` event that you see is actually exactly 4 hours from the `ChangeBasalTempPercent` event that signifies your temp basal starting.  You start to believe that the pump is lying to you.  You had assumed that a temp basal percent actually changed the rate according to your schedule, but the data seems to indicate that it sets it once based on when the temp starts and it never changes after that.

Well, hold your horses there.  It turns out that the pump actually ***does*** do what you think.  It does change the rate according to your schedule, the pump/CareLink just doesn't inform you of such endeavors.  This has been confirmed by Medtronic.  When you look at your basal rates in CareLink, they are actually going back and recomputing what your basals must've been based on the schedule that they have for that time period.  So, if you want to know the difference, you must do the same thing.

It would be much nicer if CareLink/the pump were to always emit `BasalProfileStart` events, even if a temp basal is overriding it at that point in time.  At Tidepool, we would prefer for the `BasalProfileStart` events to be the exact same as if there wasn't a temp basal (and we just apply the temp basal to what is emitted).  Though, we can also see benefit to the `BasalProfileStart` events that are affected by temp basals to have two data points:

1. The actual delivered basal rate
2. The scheduled delivered basal rate
