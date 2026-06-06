
System 1 - Christiana
- camera (for each)
- ML detection outputs bounding box and score
- camera knows its id, position and absolute orientation vector (angles)
- camera must calculate absolute orientation vector of the ray to each detected object
- system outputs a stream of data objects 
 - cam id, timestamp in utc, list of detected obj + respective orientation vector
- streams by sending regular POST


System 2 - Filipp
- processing
- gets input POSTs from cameras -> upserts to DB + caches most recent N
- has to figure out which detected objects are the same one by calculating the intersections between each ray to the detected object -> if there is an intersection withing a reasonable distance, this is the same detected object and we the itnersection is its exact position 
- system 2 then upserts the positions in to a position table with the timestamp of detection


System 3 - Bogdan
- polls detected positions from DB
- containes map with restricted areas
- if a detected position is inside a restricted area -> mark target and assign id -> upsert to DB
- display in UI, dashboard
- enable user interaction for designation of areas

