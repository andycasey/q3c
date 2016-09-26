# Q3C
Author: Sergey Koposov, Institute of Astronomy, Cambridge, UK 
Copyright (c) 2004-2016 Sergey Koposov
Email: koposov@ast.cam.ac.uk

Idea and implementation by Sergey Koposov with help from Oleg Bartunov.

Fresh versions of the software could be obtained here: https://github.com/segasai/q3c 

To read more about the Q3C indexing, please paper on Q3C from ADASS conference proceedings 
http://adsabs.harvard.edu/abs/2006ASPC..351..735K
The Citation is  "Koposov, S., & Bartunov, O. 2006, Astronomical Society of the Pacific Conference Series, 351, 735"
If you use Q3C you are kindly asked to cite this paper. Also, I 
would be also happy to hear about any usage of Q3C.

## Prerequisites

To work with Q3C you will only need to have a PostgreSQL database installed (version 9.1
or later). If you have PostgreSQL version lower than 9.1 your will need an older version 
of Q3C (1.4.x). 
For the installation of PostgreSQL refer to the PostgreSQL manual.


## Installation

1) make 
2) make install
3) Do "create extension q3c" in PostgreSQL shell

Note: To successfully compile Q3C you need to have the program called 
pg_config in your PATH

After the installation you will have several new functions in PostgreSQL.
All names of these functions start with the "q3c_" prefix.

## Table preparation for Q3C

To begin use Q3C for searches and cross-matches you should create the indexes
on the tables.

Suppose you have the table called "mytable" with "ra" and "dec" columns 
(right ascension and declination in degrees).

First, you should create the spatial index, using the following command:

`my_db# CREATE INDEX ON mytable (q3c_ang2ipix(ra, dec)); `

After that it is better to cluster that table using that index. 
Clustering procedure is the procedure of ordering the data on the disk
according to the Q3C spatial index values. If the data is already loaded
in the Database ordered by some spherical zones or something similar,
the clustering step can be ommited (although still recommended). The
clustering step may take a while if your dataset is large

`my_db# CLUSTER q3c_mytable_idx ON mytable;`

Now you should do the last step -- analyze your table:

`my_db# ANALYZE mytable;`

Now you should be able to use q3c queries.


## Q3C functions

*IMPORTANT* Throughout q3c it is assumed that all the angles are in degrees. 

The functions installed by Q3C are: 

- q3c_ang2ipix(ra, dec) -- returns the ipix value at ra and dec

- q3c_dist(ra1, dec1, ra2, dec2) -- returns the distance in degrees between 
(ra1,dec1) and (ra2,dec2)

- q3c_join(ra1, dec1, ra2, dec2, radius)  -- returns true if (ra1, dec1)
  is within radius spherical distance of (ra2, dec2). It should be used when 
  the index on q3c_ang2ipix(ra2,dec2) is created.

- q3c_ellipse_join(ra1, dec1, ra2, dec2, major, ratio, pa) -- like
  q3c_join, except (ra1, dec1) have to be within an ellipse with
  major axis major, the axis ratio ratio and the position angle pa
  (from north through east)

- q3c_radial_query(ra, dec, center_ra, center_dec, radius) -- returns
  true if ra, dec is within radius degrees of center_ra, center_dec. 
  This is the main function for cone searches. 
  function should be used if when the index on q3c_ang2ipix(ra,dec) is created)

- q3c_ellipse_query(ra, dec, center_ra, center_dec, maj_ax,
						axis_ratio, PA ) -- returns
  true if ra, dec is within the ellipse from center_ra, center_dec.
  The ellipse is specified by major axis, axis ratio and positional angle.
  function should be used if when the index on q3c_ang2ipix(ra,dec) is created)

- q3c_poly_query(ra, dec, poly) -- returns true if ra, dec is within
   the postgresql polygon poly.

- q3c_ipix2ang(ipix) -- returns a 2-array of (ra,dec) corresponding to ipix.

- q3c_pixarea(ipix, bits) -- returns the area corresponding to ipix at level
	bits (1 is smallest, 30 is the cube face) in steradians.

- q3c_ipixcenter(ra, dec, bits) -- the function returning the ipix value of the
	pixel center of certain depth covering the specified (ra,dec)

- q3c_version() -- returns the version of Q3C installed


## Query examples

- The cone search (the query of all objects within the circular region of
the sky):

To query all objects lying within radius0=0.1 deg from ra0 = 11deg, 
dec0 = 12deg in table mytable you should do:

```
my_db# SELECT * FROM mytable WHERE q3c_radial_query(ra, dec, 11, 12, 0.1);
```

The order of arguments is important here, so that the column names would come 
first, and the location where you search after, otherwise the index won't be used.

There is also another way of doing cone searches which may be appropriate if the
table that you are working with is small. In that case q3c_radial_query may 
be too CPU heavy. So you may want to query the table:

```
my_db# SELECT * FROM mytable WHERE q3c_join(11, 12, ra, dec, 0.1);
```

- The ellipse search: search for objects within the ellipse from a given point:
```
my_db=# select * from mytable WHERE
	q3c_ellipse_query(ra, dec, 10, 20, 1, 0.5 ,10);
```
returns the objects which are within the ellipse with the center at (ra,dec)=(10,20)
major axis of 1 degree, axis ratio of 0.5  and positional angle of 10 degrees.

- The polygonal query (the query of the objects which lie inside the region
  bounded by the polygon on the sphere)
  
  To query the objects in the polygon ((0,0),(2,0),(2,1),(0,1)) )
  (this is the spherical polygon with following vertices:
  (ra=0, dec=0) ; (ra=2, dec=0); (ra=2, dec=1); (ra=0, dec=1)):
  ```
my_db# SELECT * FROM mytable WHERE
		q3c_poly_query(ra, dec, '{0, 0, 2, 0, 2, 1, 0, 1}');
```
- The ellipse query (the query of objects which lie inside an ellipse, with
  	ra=10, dec=20, major axis=1deg, axis ratio=0.5 and positional angle=10deg)
  
  ```
my_db=# select * from mytable WHERE
	q3c_ellipse_query(ra, dec, 10, 20, 1, 0.5 ,10);
```

- The positional cross-match of the tables:
  
  Lets Assume that we have a huge table "table2" with ra and dec columns and
  an already created Q3C index and the smaller table "table1" with ra and dec 
  columns.
  
  Now, if we want to cross-match the tables "table1" and "table2" by position 
  with the crossmatch radius of 0.001 degrees, we should do:
  
  ```
my_db# SELECT * FROM table1 AS a, table2 AS b WHERE
		q3c_join(a.ra, a.dec, b.ra, b.dec, 0.001);
```
  
  The order of arguments is important again, because it determines whether an
  index is going to be used or not. The ra,dec columns from the table with the 
  index should go after the ra,dec columns from the table without the index.
  
  If every object in the table1 have his own error circle (like with X-ray data
  for example) (suppose that the radius of that circle is the column "err"),
  then you can run the query:
  
  ```
my_db# SELECT * FROM table1 AS a, table2 AS b WHERE
		q3c_join(a.ra, a.dec, b.ra, b.dec, a.err);
```

- The positional cross-match of the tables with the ellipse error-area:
  (for example if you want to find all the objects from one catalogue which lies
  inside the elliptical bodies of the galaxies from the second catalogue)

  It is possible to do the join when the error area of each record of the 
  catalogue is an ellipse. Then you can do the query like this
  ```
my_db# SELECT * FROM table1 AS a, table2 AS b WHERE
		q3c_ellipse_join(a.ra, a.dec, b.ra, b.dec, a.maj_ax
		a.axis_ratio, a.PA);
```
  where axis_ratio is the column with axis ratio of the ellipses and PA is the 
  column with the positional angles of them, and maj_ax is the column with major
  axises of those ellipses.
  
- The density estimation of your objects using pixelation depth of 25:
  
```
my_db# SELECT (q3c_ipix2ang(i))[1] as ra ,(q3c_ipix2ang(i))[2] as dec ,c,
				q3c_pixarea(i,25) as area from 
					(select q3c_ipixcenter(ra,dec, 25) as i, count(*) as c from
						mytable group by i) as x;
```
  returns the list of ra,dec of the Q3C pixel center, number of objects
  within a given pixel, and pixel area. If you use that query you should
  keep in mind that Q3C doesn't have the property of uniform pixel areas (as 
  opposed to HEALPIX).

- Nearest neighbor queries: 

  This query selects the only nearest neighbor for each row in your table. If there are no neighbor, the columns are filled with nulls.
```
my_db# SELECT  t.*, ss.* FROM mytable AS t,
       LEFT JOIN LATERAL (
               SELECT s.* 
                    FROM 
                        sdssdr9.phototag AS s
                    WHERE
                        q3c_join(t.ra, t.dec, s.ra, s.dec, 1./3600)
                    ORDER BY
                        q3c_dist(t.ra,t.dec,s.ra,s.dec)
                    ASC LIMIT 1
               ) as ss ON true;
```
  The idea is very simple for every row of your table mytable LATERAL() executes the "subquery" orders them by distance and limit by 1.
  
  If you want only the objects which do have the neighbors then the query will look like that
  
  ```
my_db# SELECT  t.*, ss.* FROM mytable AS t,
       LATERAL (
               SELECT s.* 
                    FROM 
                        sdssdr9.phototag AS s
                    WHERE
                        q3c_join(t.ra, t.dec, s.ra, s.dec, 1./3600)
                    ORDER BY
                        q3c_dist(t.ra,t.dec,s.ra,s.dec)
                    ASC LIMIT 1
               ) as ss ;
```

-  Nearest neighbor 2 

  This query selects the only nearest neighbor for each row in your table. If there are no neighbor, the columns are filled with nulls. This query requires presence of some object id column with the index on the table.
  ```
my_db# WITH x AS (
      SELECT *, ( SELECT objid FROM sdssdr9.phototag AS p WHERE q3c_join(m.ra, m.dec, p.ra, p.dec, 1./3600)
                  ORDER BY q3c_dist(m.ra, m.dec, p.ra, p.dec) ASC LIMIT 1) AS match_objid  FROM mytable AS m 
          )
    SELECT * FROM x, sdssdr9.phototag AS s WHERE x.match_objid=s.objid;
```


