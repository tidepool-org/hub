Release Notes
=========

This document contains release notes for Tidepool's public releases, in reverse chronological order. 

# Applications

## Blip

### v0.0.3, 27 Feb 2014

This is where you should test: https://blip-devel.tidepool.io/

You need to upload new data to see some of the improvements. See the note below about Dexcom. 

- [x] Log in with no session memory
- [x] No careteam management
- [x] Upload Carelink and Dexcom
     + Carelink seems to always be successful
     + But there seems to be an issue with large Dexcom files
     + To navigate back to the upload button if the first upload fails, click on the username.
- [x] Feedback that upload is happening
- [ ] When upload is done, taken to a screen that shows you data
     + Not yet implemented, after closing data upload window, user must click on patient data to go to data view
- [x] Ok that both dexcom and carelink/diasend must be uploaded simultaneously
- [x] Ok that upload process occurs in separate window and it is ok if the user is unable to do other activities during upload
- [x] Visualize in Tideline
  - [x] cbg
  - [x] smbg
  - [x] carbs
  - [ ] bolus
  - [ ] basal
- [x] No messages
- [ ] Best effort basal rates, expectations is that it will be buggy  
- [ ] Ok for some bolus and basal details to be missing. (It's all missing)
- [x] Stats: minimally working version, lack of bolus and basal data notwithstanding; doesn't yet deal with corner cases (i.e., you'll get NaN's etc.)
(NB: lack of basal and bolus data currently causes a lot of junk (Error: Problem parsing d="MNaN...") in the console because the stats widget chokes trying to render pie charts on no data)


# Services

