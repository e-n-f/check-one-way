check-one-way
=============

This is a tool for comparing GPS logs against OpenStreetMap extracts
to determine which streets' one-way tagging is wrong.

How to use it
-------------

First, download your city's OSM extract in XML from
[Michal Migurski's Metro Extracts](http://metro.teczno.com/).
For example, for the San Francisco Bay Area:

    $ curl -O http://osm-extracted-metros.s3.amazonaws.com/sf-bay-area.osm.bz2

Then flatten out the ways to a text file:

    $ make flatten
    $ bzcat sf-bay-area.osm.bz2 | ./flatten -s0 > sf-bay-area.flat

There will probably be lots of warnings like

    warning: couldn't find node 2335453272 for way 224753010

about missing nodes for ways that are at the edge of the bounding box for the area.

Then also flatten out your GPX files. If, for example, they are in a directory
named <code>gps</code>, do

    $ find gps -name '*.gpx' -print0 | xargs -0 ./togeo > gps.geo

Then you can match the GPS points against the OSM ways:

    $ cat gps.geo | ./match-points sf-bay-area.flat > gps.matched

And then tally up how likely it is that each street is one way, based on what
happens in the GPS logs:

    $ cat gps.matched | ./tally-oneway > gps.tally

What the results look like
--------------------------

The lines in <code>gps.tally</code> will look like this:

    ok    100.0000 yes http://www.openstreetmap.org/browse/way/55212616 38.336 56775 56775
    wrong -93.6669 unknown http://www.openstreetmap.org/browse/way/6405519 37.833 -79723 85113
    ok    100.0000 yes http://www.openstreetmap.org/browse/way/28393865 37.677 19969 19969

In this case, the one that is marked "wrong" isn't actually wrong; I just travel it
much more frequently in one direction than in the other. But those are the ones
that you should watch for.

The lines in the file are ordered from the most frequently traveled to the least,
cutting off at streets that have only been traveled 10 times. In the case of the "wrong" line here,
the fields mean:

  - <code>wrong</code>: The travel on this street doesn't match the OSM tags.
  - <code>-93.6669</code>: Travel is 94% biased away from the order of nodes in the OSM way.
    0% would mean that there were equal amounts of travel in both directions.
  - <code>unknown</code>: This way isn't currently tagged with any one-way information in OSM.
    Alternatives are <code>yes</code>, <code>no</code>, and for ways that are one-way against
    their node direction, <code>-1</code>.
    See [the OSM wiki page about one-way tagging](http://wiki.openstreetmap.org/wiki/Key:oneway).
  - <code>http://www.openstreetmap.org/browse/way/6405519</code>: This is the ID of this way in OSM
    so you can find it and fix it.
  - <code>37.833</code>: The GPS logs record 37.8 trips along the length of this way.
  - <code>-79723</code>: The balance of travel direction is 79723 feet against the current direction of the way.
  - <code>85113</code>: There are 85113 feet of total travel recorded on this way.

Searching for missing streets
-----------------------------

What's left after you remove all the known streets is a good clue to what's missing from the map.
You can use [datamaps](https://github.com/ericfischer/datamaps) to draw a mask layer that leaves
only these areas visible on the aerial photo so that it is easier to see whether they are
missing streets or just GPS noise.

To extract the missing streets into a datamaps file, you can do

    $ cat gps.matched | ./draw-unmatched | ../datamaps/encode -m8 -z20 -o unmatched.shape

and then in the script that serves the tiles, invoke datamaps rendering with

    render -g -m -t 0 -c 000000 -l2 -B 12:.12:2 unmatched.shape $z $x $y

to leave transparent an area around 100 feet wide around each GPS trace, with the rest masked to black.
