This doc is an attempt to document the issues we have run into when dealing with medtronic carelink records.

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

These events all come from the same patient action, but we must infer the relationship based on the ordering of the `Raw-Seq Num` field across equivalent `Raw-Upload ID` and `Raw-Device Type` values.  It would be a lot easier to manage the data if there were some key that the device attached to all events that came from the same action.  Something like a `Raw-Join Key` that you could just look for equality and know that the events are correlated.

### Carelink data doesn't come back in event order

There are two orderings to carelink data.  The timestamp order and the `(device,uploadId,seq_num)` order described when talking about correlating Bolus records above.  The `(device,uploadId,seq_num)` ordering is more interesting from a data correlation standpoint, but the timestamp order is more interesting from a user visualization standpoint.

Because of the out-of-order delivery of events, at tidepool, we've decided to set a "lookahead buffer" of 100 events.  When we are looking to correlate events, we look in the future 100 events for a correlation and if we do not find it, then we assume it does not exist.  This is good enough for 99.9% of data, but it does mean that it is possible we don't correlate some events that should be correlated.

This wouldn't be an issue if there were a `Raw-Join Key` field on the data as described above.  And that would be the prefered solution for this issue.  But, short of that, it would be nice if the timestamps were such that you could order based on the tuple `(timestamp,device,uploadId,seq_num)` and get a well-ordered sequence of events such that correlated events happen in order.
