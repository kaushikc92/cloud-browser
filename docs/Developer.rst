.. _developer:

****************
Developer Manual
****************

Overview
========

This projected is created using Django python for the backend. The Django ORM maps to a Sqlite database. Leaflet JS is
used to create the front end for a map view. Pandas is used to read csv files into memory and convert them to html and
the wkhtmltopdf package is used to convert html strings into images.

Components
============

This tool mainly consists of a tiler module and a map UI module. The tiler is responsible for creating the tiles for 
each csv files and returning appropriate tiles when a request is made by leafletJS. The MapUI is responsible for 
displaying the csv files as a map and for other UI related tasks such as managing the list of uploaded files and the 
space used on disk.

Map UI
-------

The map UI uses leaflet JS in order to render csv files in the form of a map. The user can upload a csv file or select
from a list of uploaded csv files. This component opens the map display for a selected file and makes requests for tiles
depending on the zoom level and the position within the map. A scroller located at the right of the map window allows
users to quickly navigate large csv files.

When a file is selected, MapUI checks if the tiles for this file are present on disk. If the tiles are not present, a
check is carried out to ensure that overall disk usage is below a soft threshold and then a request is made to the tiler
component to carry out the tiling for the selected csv. The number of rows and columns in the file are also retrieved in
this process as they need to be displayed to the user.

A dictionary of last access times for each csv file is maintained by the component. If the tiles directory exceeds the
threshold value, then MapUI starts deleting tiles pertaining to csv files in least recently accessed order until disk
usage drops below the threshold value. Although the tiles are deleted, the CSV file itself and a database entry for it
are maintained by MapUI.

Tile Creation
-------------

The tiler breaks large csv files into a number of subtables. Each subtable image is converted into an html string. This
html string is then converted into an image after setting css styles as required. In order to efficiently process large
csv files, the above procedure is carried out using multi-threading. As soon as the first subtable has been converted to
an image, the user can start browsing the table. 

Tiles requested by leaflet JS need to have dimensions of 256 * 256. The cloud browser displays five zoom levels for each
file. At the highest zoom level, the images are displayed without any resizing. However, as we go down zoom levels, a
larger portion of the images needs to be fitted into each tiles. The highest zoom level is labelled level 10 and the
lowest zoom level is labelled level 6. As we go from zoom level 10 to level 9, a 512 * 512 image needs to be compressed
into a 256 * 256 level 9 tile. The subtable images are adjusted such that their dimensions are multiples of the largest
tiles, i.e. level 6 tiles. This is done by concatenating an appropriate portion from the top of the next subtable image
to the current one and padding the right side of the subtable image. For the last image, the bottom is padded.  

A database entry is added for each subtable image. The database entry consists of the csv file it belongs to, the
subtable number and the total number of level 10 tiles in the image along each axis.

The tiling process is carried out using multi-threading in order to speed up the process for very large csv files.
After the first subtable image has been created, the user can start browsing the csv file. A background thread is used
to further spawn a batch of threads which convert subtables to html and subsequently to JPEG images. This batch of
images then have their sizes adjusted as described earlier, have database entries added for each of them and then are 
put into a write queue in order to be written out to disk. A batch of worker threads continuously pull out tasks from
the write queue and perform them. The task in this instance refers to writing a subtable image to disk.

The number of rows per subtable is decided based on maximum number of characters within each column and the total
number of columns. We allow a maximum of 2000 characters per cell which is roughly half a page of text, beyond which
truncation occurs. Tradeoffs occur while selecting an appropriate number of rows per subtable becuase a large number of
rows make it impossible to convert the subtable into an image in the proceeding steps. Conversely, a small number of rows
results in less efficient storage and processing of tiles.  

Tiling Service
--------------

The Tiler module responds to tile requests made by Leaflet JS. Each request consists of tile x,y coordinates and the
zoom level. The tiler carries out a boundary check to ensure the corresponding tile is available and then calculates the
subtable image that it belongs to. For this, it uses the database entries which contain the number of tiles per row and
per column for each subtable image. For zoom levels other than level 10, equivalent x,y coordinates at level 10 are
calculated. For example, (1,2) at level 9 will be (2,4) at level 10. 

The tiling service is an in-memory tiling service i.e. the tiles are not written out to disk. Each time a tile is 
requested, the appropriate tile is cut out from the corresponding subtable image and returned as a HTTP Response object.
Subtable images can be large and it would be time consuming to read in an entire subtable image from disk for each tile 
request. Furthermore, neighboring tiles usually belong to the same subtable image or at worst a neighboring subtable
image i.e. at a time leaflet JS will make requests for tiles from a maximum of 2 subtable images.

In order to combat the above isssue, a subset of subtable images are stored in memory. Subtable images are paged out
from this collection according to a least recently used policy as requests are made to tiles belonging to subtable
images not already present in memory. Django spawns a new thread for each request, hence it is likely that when the user
browses a portion of the csv file corresponding to a subtable image not many present in memory, multiple requests will
be made to read in this subtable image into memory. A condition variable is used to ensure that only one thread reads
the image into memory and the other threads wait for this thread. Once the first thread has completed, the other threads
use the subtable image now available in memory in order to cut out the appropriate tile and return it.

Design Decisions
=================

Initial iterations required all the subtable images to be created before the user could begin browsing the csv file.
This was because database entries were made per zoom level for total number of tiles rather
than per subtable image. It was not possible to know this until all subtable images had been written to disk, and hence
start up time was much larger. This was rectified by having per subtable database entries. When a new request comes in,
a bit more processing is required now in order to identify the subtable number that a tile belongs to, but this is still
fast enough that the user does not notice the latency. 

Previous iterations also had all tiles for all levels written to disk. This had two drawbacks, firstly, a large amount
of space was used up on the disk, and secondly, it took a long time to write such large amounts of data to disk. This
also interfered with incoming tile request processing threads. Hence, the decision was taken to carry out tiling in
memory. This was initially found to be slow but optimizations such as storing subtable images in memory and using a
condition variable to ensure smooth concurrent request processing resulted in much improved performance and lower disk
usage footprint.

Large subtables can result in the subtable images exceeding the maximum dimensions allowed for a JPEG file. Hence, an
algorithm for choosing number of rows per subtable image was required. Also, if sufficient width was not provided for
the html table, individual rows could wrap around for multiple lines and thus the resulting image could still exceed
maximum JPEG height, despite choosing a modest number of rows per image. However, providing excess width per image would
result in unnecessarily large JPEG images and reduce the readability of the csv as cell widths could exceed beyond a
page. A heuristic approach was chosen to ensure that maximum cell width was roughly half a page, beyond which text
wrapped around to the next line and at the same time the number of rows were chosen such that the resulting JPEG did not
exceed maximum image height.

A scroller was added in order to allow users to quickly traverse large csv files.
