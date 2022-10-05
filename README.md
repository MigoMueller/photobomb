# photobomb
Asteroid sizes from serendipitous imaging ("photobombing") of known asteroids by JWST MIRI.

Requires:
* JWST pipeline (pip install jwst)
* NEATM: https://github.com/MigoMueller/NEATM
* astroquery (contained in jwst pipeline?)

## Version history
* 2022/10/05: first uploads

## Approach
* ID known asteroids intersecting the MIRI imager FOV during an observation, done in three steps.  These are designed to be as efficient as possible, keeping in mind that the pipeline (especially level 1) is computationally expensive, and queries to the small-body ID tool are slow.  Horizons queries are much faster.  Make a point of only downloading MIRI data that are actually needed.

  1. Input: PID and observation number, MAST. Retrieve start and end time, and FOV center from MAST (without downloading data just yet--astroquery supports this, I believe).  Run JPL "Small-Body Identification Tool" (https://ssd.jpl.nasa.gov/tools/sb_ident.html#/).  These queries are slow (~1min per call), so be generous with search radius (10'?).  We want to be sure to catch all intersecting asteroids, including quick NEOs, in a single call.  False positives are OK; they'll be weeded out in following steps.  This step to be adapted from Lenka's work.  Change: URL (and API?) has changed.  Save output in ASCII files out/pid/obs/candidates1.csv (or json?).  Include timestamp (start of query in UTC), FOV center, start time, end time.  Then: one line/item per asteroid candidate. Asteroid line: unique identifier (compressed designation?) + timestamp when it was first identified.  If output file already exists, make sure not to just overwrite it.  If candidate has been found before, don't touch its line.  If candidate is no longer found but was found before: not sure what to do then (delete from file? bug out?).  This is to avoid unnecessary pipeline-runs downstream.

  2. Input: candidates1 from above.  For all candidates, get Horizons ephemeris for obs start time and end time and ~100 (value TBD) times inbetween.  Calculate min distance FOV center, orbit. Reject candidate if min distance > a TBD "FOV radius" (radius of the smallest circle containing the FOV?) or if orbital uncertainty > some value.  Output retained candidates into ASCII file candidates2 analogous to candates1 above.  Again, take care of timestamps: if candidate was found before, do not update timestamp.
  
  3. Input: candidates2 from above.  If candidates2 contains at least one asteroid, download rate (or cal?) file from MAST (unless it's present locally, already).  Get Horizons ephemeris for start and end time.  For n~100 (value TBD) times inbetween, convert RA/dec to x,y using WCS in data file.  Retain if at least one point intersects FOV (minus LRS?).  Output candidates3, analogous to candidates 1 and 2, with extra columns motion rate x/y in pixels/frame.  Again, take care of timestamps.  [Error mode: if motion rate has change appreciably between various runs of the tool.  Shouldn't be a problem in practice, I believe]
  
* Shift charges: download raw (uncal) data if not present and if candidates3 contains any asteroids.  Create one sub-dir per asteroid.  Shift charges using routines from Stella's work by the motion rates in candidates3.  Save shifted uncal file in asteroid directory.  Create log in asteroid directory: log asteroid desig and timestamp of Horizons query, motion rates.

* JWST pipeline: run pipeline levels 1,2,3 in one script each, save output. Add to log: timestamp, jwst.__version__, CRDS context or some such metric of CRDS version.  The idea is we should be able to re-run, say, the level 2 pipeline if there's any relevant cal updates without redoing the prevous work.  TBD if level 3 outputs asteroid flux straight-away.  Check imager tutorial notebooks.  Output asteroid flux with observation mid-time.  Color corrections?

* NEATM: simple-minded implementation would fit fixed-eta NEATM to each single detection.  There will be more than one, though, in general: dithers and/or observations at different wavelengths contained in the same PID.  TBD how those get combined before NEATM fitting.


## Code heritage

For development, use notebooks in directory notebooks.  Once everything works, develop scripts that can be run in a cron job or similar, in subdir scripts.

This repo is partly based on work by students under my supervision: 
* https://github.com/lenkaaaa49/Serendipitous-asteroid-observation
* https://github.com/StellaTsilia/MIRI-asteroid-detection
