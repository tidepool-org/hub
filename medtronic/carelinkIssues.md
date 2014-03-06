This doc is an attempt to document the issues we have run into when dealing with medtronic carelink records.

### `Dual/Square` bolus events can exist without a corresponding `Dual/Normal`.

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
