
System 1
- camera (for each)
- ML detection outputs bounding box and score
- camera knows its id, position and absolute orientation vector (angles)
- camera must calculate absolute orientation vector of the ray to each detected object
- system outputs a stream of data objects 
 - cam id, timestamp in utc, list of detected obj + respective orientation vector
- streams by sending regular POST


System 2 
- processing
- gets input POSTs from cameras -> upserts to DB + caches most recent N
- has to figure out which detected objects are the same one by calculating the intersections between each ray to the detected object -> if there is an intersection withing a reasonable distance, this is the same detected object and we the itnersection is its exact position 
- system 2 then upserts the positions in to a position table with the timestamp of detection


- `MAP`
- `[DW?] sequence of poi`
- `- map with restricted areas`
- `-> mark target`
- `-> UI, dashboard`
- `-> user interaction for`
- `designation of area`
:w
sd
Right side:

- `Y SIH` or similar unclear shorthand

## What it seems to be about

These notes appear to be brainstorming for a mapping / vision / tracking system. Topics mentioned include:

- machine learning / object detection
- output streams
- angle vectors to a drone
- bounding boxes
- caching and processing
- triangulation / position estimation
- map UI / dashboard work
- marking targets and restricted areas
- user interaction for defining areas

## Confidence

Several words are unclear because the handwriting is faint. The transcription above is a best effort and may need manual correction.
