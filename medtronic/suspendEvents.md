This page discusses Tidepool's current understanding of the semantics of various suspend events seen from Carelink.  It is the result of reading the user manual, making assumptions and reverse engineering because we have not yet contacted someone at Medtronic to confirm or deny that these are the semantics of the events.

A suspend event generally looks something like:

```
15942,3/14/14,05:48:38,3/14/14 05:48:38,,,,,,,,,,,,,Suspended,,,,,,,,,,,,,,,,,ChangeSuspendEnable,"ENABLE=user_suspend, ACTION_REQUESTOR=rf_diagnostic, PRE_ENABLE=normal_pumping",12002904701,12345678,102,Paradigm Veo - 723
```

The important bits of information with definitions of terms used in the rest of this document are

1. ***timestamp***: the timestamp
2. ***event type***: The `Raw-Type` label, `ChangeSuspendEnable`
3. ***status***: In the `Raw-Values` field, the `ENABLE` key (it has value `user_suspend` here)
4. ***previous status***: In the `Raw-Values` field, the `PRE_ENABLE` key (it has value `normal_pumping` here), only exists on newer pumps (known to not be reported by the Revel)

We enumerate the suspend and resume events based on their status.  It is important to note that the same set of values can appear in both `status` and `previous status`.

## Suspend events

When ***any*** of these events is seen, it means that the pump is suspended.  It is possible to see multiple of these before a resume event is seen.  When that happens, the multiple events are simply state changes internal to the pump and do not reflect changes in user-facing functionality (i.e. the pump is suspended).  Therefore, the time range that the pump was suspended is determined by the first suspend event seen followed by the first resume event seen.

When processing these events, previous status is very helpful to determine when a state change (to/from suspension) actually occurred, but it is important to keep in mind that support for the field only exists on some pumps.  Or, so one might think.  However, we have seen at least one case out in the wild where a pump apparently generated "duplicate" suspend events with different timestamps, but the exact same status and previous status.  So, this is not actually reliable.

### `user_suspend`

This indicates that the user manually told the pump to suspend.  This can happen when the user wants insulin to stop being delivered, but it also often happens when uploading new data to carelink as well as during some routing operations like insertion set changes.

### `alarm_suspend`

This means that an alarm on the pump has caused a suspend to occur.  This is automated and does not signify any user interaction.

### `low_suspend_mode_1`

This means that the pump is suspending operation due to low glucose.  This is automated and does not signify any user interaction.  It generally follows an `alarm_suspend`.

### `low_suspend_no_response`

This means that the user has not responded to an `alarm_suspend`.  This is automated and does not signify any user interaction

### `low_suspend_user_selected`

This indicates that the user has decided to allow the automated suspend to persist.  That is, when `alarm_suspend` happens, an alarm is shown to the user and they have the choice of canceling or allowing the suspend.  When they make the choice to maintain the suspend, this event occurs.

## Resume events

When ***any*** of these events is seen, it means that the pump has resumed operation.  It is possible to see multiple of these in succession, as with Suspend events, extra events can be ignored.

### `normal_pumping`

This is the bread-n-butter "resume" status.  It means that the pump is operating normally without any caveats.  It can happen due to a user manually resuming the pump, or it can be a subsequent event from one of the other resume events.

### `auto_resume_complete`

This means that the pump automatically resumed operation without user interaction.

### `auto_resume_reduced`

This means that the pump automatically resumed operation without use interaction.  There is no indication of what that reduced dose might mean, and the "human readable" version of the status that Carelink provides is "Suspend (auto resume 100% Basal)" which seems to indicate that it's not reduced at all.

### `user_restart_basal`

This means that the user manually restarted their basal operation after an automated suspend.  I.e. after `alarm_suspend` the user cleared the alarm and choose to restart their basal rate.
