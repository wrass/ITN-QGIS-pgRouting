# Setting up ITN for use with pgRouting in QGIS
This document is intended as a guide to loading Ordnance Survey Mastermap ITN into PostGIS for use in QGIS. As my area of interest is Wales, this will take into account both Welsh and English language information.

Some sections of this guide are based on work by Ross McDonald (https://github.com/mixedbredie/itn-for-pgrouting) and SQL views from Astun Technology's Loader (https://github.com/AstunTechnology/Loader/blob/master/extras/ordnancesurvey/osmm/itn/views.sql).

## Software Pre-Requisites
- [PostgreSQL 9.4.8](https://www.postgresql.org/)
- [PostGIS 2.2.2](http://postgis.net/)
- [QGIS 2.16.0-Nødebo](http://qgis.org/)
- [OS Translator II 1.2.4](http://www.lutraconsulting.co.uk/products/ostranslator-ii/)

This guide assumes that you have access to a locally stored copy of the raw ITN .gz files from Ordnance Survey and that you've already set up a PostGIS connection in QGIS.

## Loading ITN Data
As standard, OS Translator II v1.2.4 doesn't include some bits of information that we want to include in the final network build, so it'll need a couple of tweaks before starting the translation.  First edit the .gfs used for ITN - normally located at `%userprofile%\.qgis2\python\plugins\OSTranslatorII\gfs\OS Mastermap ITN (v7).gfs` in Windows - and insert the following after line 603:
``` XML
    <PropertyDefn>
      <Name>environmentqualifier_classification</Name>
      <ElementPath>environmentQualifier|classification</ElementPath>
      <Type>StringList</Type>
    </PropertyDefn>
    <PropertyDefn>
      <Name>environmentqualifier_instruction</Name>
      <ElementPath>environmentQualifier|instruction</ElementPath>
      <Type>StringList</Type>
    </PropertyDefn>
```

When done, open QGIS and OS Translator II from the Plugins menu, choose OS Mastermap ITN (v7) as your input and select your PostGIS connection, set the target schema to `osmm_itn`, set the location of your ITN data and make sure Create spatial index is selected, then hit OK to start the translation. As my ITN supply was chunked I opted to use the Remove duplicates option as well.

**Note that you should set the Processor Cores option according to the number of actual _physical_ processor cores in your machine, as per the instructions at http://www.lutraconsulting.co.uk/products/ostranslator-ii/#postgis-performance-tips.**

This is likely to take a bit of time depending on the coverage of your dataset, though on my old work laptop and loading across the network to a server ITN for the whole of Wales takes a little over half an hour to load, so it's pretty quick.

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
UPDATE osmm_itn.roadlink
SET roadname = r.roadname
FROM (SELECT concat_ws(' / ', a.roadnamecy, a.roadnameen) as roadname, b.fid
      FROM osmm_itn.road a
        INNER JOIN osmm_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN osmm_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'Named Road') AS r
WHERE roadlink.fid = r.fid;

UPDATE osmm_itn.roadlink
SET roadnumber = r.roadnumber
FROM (SELECT a.roadnameen as roadnumber, b.fid
      FROM osmm_itn.road a
        INNER JOIN osmm_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN osmm_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'Motorway') AS r
WHERE roadlink.fid = r.fid;

UPDATE osmm_itn.roadlink
SET roadnumber = r.roadnumber
FROM (SELECT a.roadnameen as roadnumber, b.fid
      FROM osmm_itn.road a
        INNER JOIN osmm_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN osmm_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'A Road') AS r
WHERE roadlink.fid = r.fid;

UPDATE osmm_itn.roadlink
SET roadnumber = r.roadnumber
FROM (SELECT a.roadnameen as roadnumber, b.fid
      FROM osmm_itn.road a
        INNER JOIN osmm_itn.lu_road_roadlink lu ON a.fid = lu.road_fid
        INNER JOIN osmm_itn.roadlink b ON lu.roadlink_fid = b.fid
      WHERE a.descriptivegroup = 'B Road') AS r
WHERE roadlink.fid = r.fid;

VACUUM ANALYZE osmm_itn.roadlink;
```

## One Way Streets
One way street information needs to be pulled from the ITN roadrouteinformation so we can use it to build the road network later on. Again we'll need to create a lookup, this time between the roadlink and roadrouteinformation tables:
``` PLpgSQL
CREATE MATERIALIZED VIEW osmm_itn.lu_rri_roadlink AS
SELECT roadrouteinformation_fid,
       replace(roadlink_fid, '#', '') AS roadlink_fid,
       roadlink_order
FROM (SELECT fid AS roadrouteinformation_fid,
             unnest(directedlinkref) AS roadlink_fid,
             generate_subscripts(directedlinkref, 1) AS roadlink_order
      FROM osmm_itn.roadrouteinformation
) AS a;
CREATE INDEX lu_rri_roadlink_rri_fid ON osmm_itn.lu_rri_roadlink (roadrouteinformation_fid);
CREATE INDEX lu_rri_roadlink_roadlink_fid on osmm_itn.lu_rri_roadlink (roadlink_fid);
```

Now to create a one-way view based on this lookup:
``` PLpgSQL
CREATE OR REPLACE VIEW osmm_itn.vw_itn_oneway AS 
SELECT replace(array_to_string(rri.directedlinkref, ', '::text), '#'::text, ''::text) AS directedlink_href,
       rrirl.roadrouteinformation_fid,
       array_to_string(rri.directedlinkorientation, ', '::text) AS directedlink_orientation,
       array_to_string(rri.environmentqualifier_instruction, ', '::text) AS environmentqualifier,
       CASE WHEN rri.directedlinkorientation::text = '{+}'::text THEN 512 ELSE 1024 END AS oneway_attr
FROM osmm_itn.lu_rri_roadlink rrirl
  RIGHT JOIN osmm_itn.roadrouteinformation rri ON rri.fid::text = rrirl.roadrouteinformation_fid::text
WHERE rri.environmentqualifier_instruction = '{"One Way"}'::character varying[];
```

## Creating the Network
The following creates an empty table to hold the route network geometry and attributes required for pgRouting. We'll name the table `itn_network`:
``` PLpgSQL
CREATE TABLE osmm_itn.itn_network (
  fkey_pkey integer,
  gradeseparation_s integer,
  gradeseparation_e integer,
  oneway integer,
  toid character varying(30),
  dftname character varying(200),
  roadname character varying(200),
  length double precision,
  natureofroad character varying(40),
  geometry geometry,
  descriptivegroup character varying(20),
  descriptiveterm character varying(40),
  rl_attribute integer,
  rl_speed integer,
  rl_width integer,
  rl_weight integer,
  rl_height integer,
  gid serial NOT NULL,
  source integer,
  target integer,
  cost_len double precision,
  rcost_len double precision,
  one_way character varying(2),
  cost_time double precision,
  rcost_time double precision,
  x1 double precision,
  y1 double precision,
  x2 double precision,
  y2 double precision,
  to_cost double precision,
  rule text,
  isolated integer,
  CONSTRAINT itn_network_pkey PRIMARY KEY (gid)
) WITH ( OIDS=FALSE );

CREATE INDEX itn_network_geometry_gidx ON osmm_itn.itn_network USING gist (geometry);
CREATE INDEX itn_network_source_idx ON osmm_itn.itn_network USING btree (source);
CREATE INDEX itn_network_target_idx ON osmm_itn.itn_network USING btree (target);
```

The next step is to create a function that builds the network when required:
``` PLpgSQL
CREATE OR REPLACE FUNCTION osmm_itn.create_itn_network()
  RETURNS integer AS
$BODY$
  DECLARE cur_rl CURSOR FOR
    SELECT rl.descriptivegroup,
           rl.descriptiveterm,
           rl.fid,
           rl.roadnumber,
           rl.roadname,
           rl.length,
           rl.natureofroad,
           rl.wkb_geometry,
           rl.ogc_fid,
           ow.oneway_attr,
           rl.gradeseparationneg as gradeseparation_s,
           rl.gradeseparationpos as gradeseparation_e
    FROM osmm_itn.roadlink rl
      LEFT JOIN osmm_itn.vw_itn_oneway ow ON rl.fid = ow.directedlink_href;

    v_descriptivegroup osmm_itn.roadlink.descriptivegroup%TYPE;
    v_descriptiveterm osmm_itn.roadlink.descriptiveterm%TYPE;
    v_fid osmm_itn.roadlink.fid%TYPE;
    v_itnroadlink_dftname osmm_itn.roadlink.roadnumber%TYPE;
    v_itnroadlink_roadname osmm_itn.roadlink.roadname%TYPE;
    v_length osmm_itn.roadlink.length%TYPE;
    v_natureofroad osmm_itn.roadlink.natureofroad%TYPE;
    v_polyline osmm_itn.roadlink.wkb_geometry%TYPE;
    v_primary_key osmm_itn.roadlink.ogc_fid%TYPE;
    v_separation_s osmm_itn.roadlink.gradeseparationneg%TYPE;
    v_separation_e osmm_itn.roadlink.gradeseparationneg%TYPE;
    v_oneway osmm_itn.vw_itn_oneway.oneway_attr%TYPE;
    v_rl_attribute integer;
    v_rl_roadname varchar(150);
    v_sql varchar(120);
    v_sep_s integer;
    v_sep_e integer;
    v_date date;
    v_ctr integer;
    v_ctr_target integer;
    v_total integer;

  BEGIN
    -- truncate storage table
    v_sql := 'TRUNCATE TABLE osmm_itn.itn_network';
    EXECUTE v_sql;
    OPEN cur_rl;
    SELECT 'now'::timestamp INTO v_date;
    v_ctr := 1;
    v_total := 500;
    v_ctr_target := count(1) from osmm_itn.roadlink;
    LOOP
      FETCH cur_rl INTO v_descriptivegroup, v_descriptiveterm, v_fid, v_itnroadlink_dftname, v_itnroadlink_roadname, v_length, v_natureofroad, v_polyline, v_primary_key, v_rl_attribute, v_oneway, v_separation_s, v_separation_e;
      EXIT WHEN NOT FOUND;
      
      -- build roadname text output
      IF ((v_itnroadlink_roadname IS NULL) AND (v_itnroadlink_dftname IS NULL)) THEN 
        v_rl_roadname := v_descriptiveterm;
      ELSIF (v_itnroadlink_roadname IS NULL) THEN 
        v_rl_roadname := v_itnroadlink_dftname;
      ELSIF (v_itnroadlink_dftname IS NULL) THEN
        v_rl_roadname := v_itnroadlink_roadname;
      ELSE
        v_rl_roadname:= v_itnroadlink_roadname || ' (' || v_itnroadlink_dftname || ')';
      END IF;
      
      -- build road attribute output including oneway
      IF v_rl_attribute IS NULL THEN
        v_rl_attribute := 0;
      END IF;
      IF (v_natureofroad = 'Roundabout')THEN
        v_rl_attribute := v_rl_attribute + 8;
      ELSIF (v_natureofroad = 'Slip Road')THEN
        v_rl_attribute := v_rl_attribute + 7;
      ELSIF (v_descriptiveterm = 'A Road') THEN
        v_rl_attribute := v_rl_attribute + 2;
      ELSIF (v_descriptiveterm = 'B Road') THEN
        v_rl_attribute := v_rl_attribute + 3;
      ELSIF (v_descriptiveterm = 'Alley') THEN
        v_rl_attribute := v_rl_attribute + 6;
      ELSIF (v_descriptiveterm = 'Local Street') THEN
        v_rl_attribute := v_rl_attribute + 5;
      ELSIF (v_descriptiveterm = 'Minor Road') THEN
        v_rl_attribute := v_rl_attribute + 4;
      ELSIF (v_descriptiveterm = 'Motorway') THEN
        v_rl_attribute := v_rl_attribute + 1 + 64;
      ELSIF (v_descriptiveterm = 'Pedestrianised Street') THEN
        v_rl_attribute := v_rl_attribute + 9 + 128;
      ELSIF (v_descriptiveterm = 'Private Road - Publicly Accessible') THEN
        v_rl_attribute := v_rl_attribute + 10;
      ELSIF (v_descriptiveterm = 'Private Road - Restricted Access') THEN
        v_rl_attribute := v_rl_attribute + 11;
      ELSE
        v_rl_attribute := v_rl_attribute + 0;
      END IF;
      
      -- add road grade seperation
      IF v_separation_s = 1 THEN
        v_sep_s:= v_separation_s;
      ELSE
        v_sep_s:= 0;
      END IF;
      IF v_separation_e = 1 THEN
        v_sep_e:= v_separation_e;
      ELSE
        v_sep_e:= 0;
      END IF;
      
      -- insert the values into the table  
      INSERT INTO osmm_itn.itn_network(geometry, fkey_pkey, toid, dftname, roadname, natureofroad, length, descriptivegroup, descriptiveterm, rl_attribute, oneway, gradeseparation_s, gradeseparation_e, gid)
        VALUES (v_polyline, v_primary_key, v_fid, v_itnroadlink_dftname, v_itnroadlink_roadname, v_natureofroad, v_length, v_descriptivegroup, v_descriptiveterm, v_rl_attribute, v_oneway, v_sep_s, v_sep_e, v_primary_key);
      IF (v_ctr = v_total) THEN
        v_total := v_total + 500;
        RAISE INFO '% / % records inserted', v_ctr, v_ctr_target;
      END IF;
      
      v_ctr := v_ctr + 1;
    END LOOP;
    RETURN 1;
    CLOSE cur_rl;
  END;
$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
```

.. and fill the table with data:
``` PLpgSQL
SELECT osmm_itn.create_itn_network();
```
