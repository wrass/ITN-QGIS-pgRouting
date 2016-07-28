# Setting up ITN for use with pgRouting in QGIS
This document is intended as a guide to loading Ordnance Survey Mastermap ITN into PostGIS for use in QGIS. As my area of interest is Wales, this will take into account both Welsh and English language information.

Various sections are based on work by Ross McDonald, see https://github.com/mixedbredie/itn-for-pgrouting.

## Software Pre-Requisites
- [PostgreSQL 9.4.8](https://www.postgresql.org/)
- [PostGIS 2.2.2](http://postgis.net/)
- [QGIS 2.16.0-Nødebo](http://qgis.org/)
- [OS Translator II 1.2.4](http://www.lutraconsulting.co.uk/products/ostranslator-ii/)

This guide assumes that you have access to a locally stored copy of the raw ITN .gz files from Ordnance Survey and that you've already set up a PostGIS connection in QGIS.

## Loading ITN Data
Open OS Translator II from the Plugins menu in QGIS, choose OS Mastermap ITN (v7) as your input and select your PostGIS connection, set the target schema to `osmm_itn`, set the location of your ITN data and make sure Create spatial index is selected, then hit OK to start the translation. As my ITN supply was chunked I opted to use the Remove duplicates option as well.

**Note that you should set the Processor Cores option according to the number of actual _physical_ processor cores in your machine, as per the instructions at http://www.lutraconsulting.co.uk/products/ostranslator-ii/#postgis-performance-tips.**

This is likely to take a bit of time depending on the coverage of your dataset, though on my old work laptop and loading across the network to a server, ITN for the whole of Wales takes a little over half an hour to load.

## Road Names / Numbers
As each road record is linked to multiple network links via a list in the networkmember field, we'll first need to create a lookup between the road and roadlink tables. This might as well be a materialized view as the data won't be changing, plus we can add indexes for a minor storage cost vs a significant speed increase:

``` PLpgSQL
CREATE MATERIALIZED VIEW osmm_itn.lu_road_roadlink AS
  SELECT road_fid, replace(roadlink_fid, '#', '') AS roadlink_fid FROM (
    SELECT fid AS road_fid, unnest(networkmember) AS roadlink_fid FROM osmm_itn.road
  ) AS a;
CREATE INDEX lu_road_roadlink_road_fid_idx ON osmm_itn.lu_road_roadlink (road_fid);
CREATE INDEX lu_road_roadlink_roadlink_fid ON osmm_itn.lu_road_roadlink (roadlink_fid);
```

Next, we'll need roadname and roadnumber fields in the roadlink table:
``` PLpgSQL
ALTER TABLE osmm_itn.roadlink ADD COLUMN roadname character varying(250);
ALTER TABLE osmm_itn.roadlink ADD COLUMN roadnumber character varying(70);
```

Now to update the roadlink table with road names and road numbers.  These queries update each road type in turn:
``` PLpgSQL
UPDATE os_itn.roadlink
SET roadname = r.roadname
FROM (SELECT concat_ws(' / ', a.roadnamecy, a.roadnameen) as roadname, b.fid
      FROM os_itn.road a
        INNER JOIN os_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN os_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'Named Road') AS r
WHERE roadlink.fid = r.fid;
```
``` PLpgSQL
UPDATE os_itn.roadlink
SET roadname = r.roadname
FROM (SELECT a.roadnameen as roadname, b.fid
      FROM os_itn.road a
        INNER JOIN os_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN os_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'Motorway') AS r
WHERE roadlink.fid = r.fid
AND roadlink.roadname = '';
```
``` PLpgSQL
UPDATE os_itn.roadlink
SET roadname = r.roadname
FROM (SELECT a.roadnameen as roadname, b.fid
      FROM os_itn.road a
        INNER JOIN os_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN os_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'A Road') AS r
WHERE roadlink.fid = r.fid
AND roadlink.roadname = '';
```
``` PLpgSQL
UPDATE os_itn.roadlink
SET roadname = r.roadname
FROM (SELECT a.roadnameen as roadname, b.fid
      FROM os_itn.road a
        INNER JOIN os_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN os_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'B Road') AS r
WHERE roadlink.fid = r.fid
AND roadlink.roadname = '';
```
