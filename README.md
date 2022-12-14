# The Southern-Hemisphere Spectroscopic Redshift Compilation
---
## Description
This page is reserved for releases of a compilation of spectrocopic redshifts for the Southern Hemisphere (declination below 10 degrees) focused on galaxies.

This compilation contains 3717 catalogues of spectroscopic redshifts from services such as [VizieR](http://vizier.cds.unistra.fr/), [HEASARC](https://heasarc.gsfc.nasa.gov/), [CasJobs](http://skyserver.sdss.org/CasJobs/), and others. After removing duplicates, the number of catalogues in the final compination is 2093. The catalogue name in the TAP services, work titles, number of objects, and authors are present in the files [`Catalogues_All.csv`](https://github.com/ErikVini/SpecZCompilation/blob/5a47de5612b13d2589eb4168fb221f39cc3b34b7/Catalogues_All.csv) and [`Catalogues_Used.csv`](https://github.com/ErikVini/SpecZCompilation/blob/5a47de5612b13d2589eb4168fb221f39cc3b34b7/Catalogues_Used.csv) for all downloaded tables and the ones used in the final compilation (after removing duplicates) respectively.


The columns avaliable are:
* `RA`: right ascension (degrees),
* `DEC`: declination (degrees),
* `z`: spectroscopic redshift,
* `e_z`: error in the spectroscopic redshift,
* `f_z`: flag for the spectroscopic redshift quality,
* `class_spec`: spectroscopic classification of the object,
* `source`: TAP service and catalogue from which the information was obtained.

Not all catalogues used contain information about the redshift error, quality flags, and classes. In these situations the value is left empty.

## How it was done
A script written in Python is used to download all catalogues from the VizieR and HEASARC table access protocol (TAP) services using the following queries:

RA columns:
```
SELECT t.table_name, c.column_name AS ra_col, c.description as ra_desc
FROM tap_schema.tables AS t
JOIN tap_schema.columns AS c USING (table_name)
WHERE c.ucd = 'pos.eq.ra;meta.main'
```

DEC columns:
```
SELECT t.table_name, c.column_name AS dec_col, c.description as dec_desc
FROM tap_schema.tables AS t
JOIN tap_schema.columns AS c USING (table_name)
WHERE c.ucd = 'pos.eq.dec;meta.main'
```

Redshift tables:
```
SELECT t.table_name, t.description, c.column_name AS zcol, c.description as zcol_desc
FROM tap_schema.tables AS t
JOIN tap_schema.columns AS c USING (table_name)
WHERE c.ucd = 'src.redshift'
```

Redshift error columns:
```
SELECT t.table_name, c.column_name AS zerr_col, c.description as zerr_desc
FROM tap_schema.tables AS t
JOIN tap_schema.columns AS c USING (table_name)
WHERE c.ucd = 'stat.error;src.redshift'
```

Redshift flag columns:
```
SELECT t.table_name, t.description, c.column_name AS zflg_col, c.description as zflg_desc
FROM tap_schema.tables AS t
JOIN tap_schema.columns AS c USING (table_name)
WHERE c.ucd = 'meta.code.error;src.redshift' OR c.ucd = 'meta.code.qual;src.redshift' -- For VizieR
WHERE c.ucd = 'meta.code;src.redshift' -- For HEASARC
```

Morphological classification columns:
```
SELECT t.table_name, c.column_name AS class_col, c.description as class_desc
FROM tap_schema.tables AS t
JOIN tap_schema.columns AS c USING (table_name)
WHERE c.ucd = 'meta.code.class' -- For VizieR
WHERE c.ucd = 'src.class' -- For HEASARC
```

These queries are used to obtain the table names, respective column names (for coordinates, redshift, redshift error, and class) and their descriptions. This information is used to generate another table which contains all information needed to query for the objects (`RA`, `DEC`, `z`, `e_z`, `f_z`, `class_spec`).

Before downloading all possible tables, and since there are duplicates, a pre-processing is done to remove tables that:
* The `z` column does not correspond to spectroscopic redshifts (such as metallicities or galaxy height)
* The `e_z`, `f_z`, or `class` columns does not correspond to the information needed
Example: some tables have flags in magnitudes, but they share the same unified content descriptors (UCDs) with spectroscopic reddshift UCD, so the table may be downloaded with the wrong information.

This process is done for VizieR and HEASARC. In both cases any tables with zero objects are removed.

For VizieR, a correction for J1950 coordinates is applied. Also, any tables that represent distances as `cz` also have the redshift and error values corrected.

## Classes

For tables that have this information, a procedure was applied to group classes into `STAR`, `GALAXY`, `QSO`, `AGN`, `UNCLEAR`, or `GLOBCLUSTER`. When avaliable, a few sub-classes are included:

* `GALAXY`
  * `GALAXY(SF)`: Star-forming galaxies
  * `GALAXY(CLUSTER)`: Cluster of galaxies
  * `GALAXY(BINARY)`: Binary galaxies
  * `GALAXY(PAIR)`: Galaxy pair
  * `GALAXY(TRIP)`: Galaxy triplet
  * `GALAXY(GROUP)`: Galaxy group
* `STAR`
  * `STAR(SN)`: Supernovae
  * `STAR(WD)`: White-dwarf star
* `QSO`
  * `QSO(BLLAC)`: BL-Lac object
* `UNCLEAR`
  * `UNCLEAR(POINT-SOURCE)`
  * `UNCLEAR(EXTENDED)`
  * `UNCLEAR(BROADLINE) `

The `UNCLEAR` class is reserved for objects where the classification was not clear enough to be included in the other five groups.

## Flags

For tables with this information, a manual verification was made in order to classify flags as `KEEP` or `REMOVE`. The `KEEP` flag indicates that the spectroscopic redshift is reliable according to the authors of the catalogue. The original flag is maintained whitin parenthesis after the `KEEP` or `REMOVE` words.

## Merging catalogues

After all VizieR and HEASARC catalogues were downloaded, they were concatenated with the SDSS DR17 catalogue obtained using the following query:
```
SELECT spec.specObjID, spec.ra, spec.dec, spec.z as z, spec.zErr as e_z, spec.zWarning as f_z, spec.class as spec_class, spec.instrument
INTO MyDB.SpecZ_DR17
FROM SpecObj as spec
WHERE
  spec.dec <= +10 AND spec.class != 'STAR'
```
For SDSS, the `source` column indicates if that observation was made with the SDSS or BOSS instruments.

We also downloaded the [PRIMUS](https://primus.ucsd.edu/index.html) catalogue, and it is also concatenated with the others.

## Duplicate removal procedure

After concatenating all catalogues, the resulting catalogue will inevitably have duplicate objects. To try and remove those objects the [STILTS](http://www.star.bris.ac.uk/~mbt/stilts/sun256/index.html) software was used.

Before removing duplicates, the table was sorted in order to keep the objects with most information at the top. The catalogue is then composed of blocks:
* Objects with `e_z`, `f_z`, and `class_spec`,
* objects with `e_z` and `class_spec`,
* objects with `e_z` and `f_z`,
* objects with `f_z` and `class_spec`,
* objects with `e_z`,
* objects with `class_spec`,
* objects with `f_z`, and
* objects without `e_z`, `f_z` or `class_spec`.
Whenever the spectroscopic redshift error was available, the objects that block were sorted according to the error value (from lowest to highest).

An internal match is done using the `Sky+X` match parameter with `RA`, `DEC` and `z` with a 2 arcsecond maximum separation in coordinates and 0.002 in redshift, keeping only the first ocurrence (thus the importance of the sorting procedure above):
```
java -jar stilts.jar tmatch1 matcher=sky+1d values='RA DEC z' params='2 0.002' action=keep1 in=InputTable.csv out=OutputTable.csv
```

## Known issues

Some redshifts are duplicated even after the previous duplicate removal procedure. This is more common for extended objects (such as big galaxies in nearby clusters). This happens because, although the measurements lie inside a 2 arcsecond radius, they differ by more than 0.002 in z, thus they are not detected as duplicated and are not removed.

## The final catalogue

![Distribution of objects in the sky. Image also available in "Images" folder.](Images/AllSky_Spec_Scatter.png?raw=true "Distribution of objects in the sky.")

![Number of objects per class. Image also available in "Images" folder.](Images/Class_Distribution.png?raw=true "Number of objects per class.")

![Number of objects per flag. Image also available in "Images" folder.](Images/Flags_Distribution.png?raw=true "Number of objects per flag.")

![Distribution of redshifts per class. Image also available in "Images" folder.](Images/SpecZ_Distribution.png?raw=true "Distribution of redshifts per class.")
