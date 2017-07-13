Building TIGER 2016 Road Tiles
==============================

US Census Bureau's TIGER dataset is one of the primary nationwide geographic datasets. Roughly 10 years ago, it was imported into OpenStreetMap and huge swaths of it haven't been touched since, even though the TIGER dataset is updated yearly. Based on earlier work that OpenStreetMap US did, [Eric Fischer's TIGER2015 layer](https://github.com/ericfischer/tiger-delta) provides an overlay that helps mappers identify roads that are missing from OpenStreetMap and gives a way to find street names for roads that might not have names in OpenStreetMap.

These instructions replicate this layer with the more recent TIGER 2016 release. The TIGER dataset includes a `ROADS` and `FEATNAMES` dataset. The `ROADS` dataset includes geometries and a `linearid` that can be joined with the `linearid` in the `FEATNAMES` dataset. In `FEATNAMES` the road names are broken into several pieces, which we expand (unabbreviate) and concatenate to form a display label. Finally, the resulting joined data is built into a mbtiles file with [tippecanoe]() and uploaded to MapBox Studio for styling.

1. Download TIGER data. `ROADS` for the geometries, `FEATNAMES` for the split-apart road names.

   ```bash
   mkdir -p ./featnames
   curl -s https://www2.census.gov/geo/tiger/TIGER2016/FEATNAMES/ | \
      grep -o '<a href=['"'"'"][^"'"'"']*['"'"'"]' | \
      grep featnames | \
      sed -e 's/^<a href=["'"'"']/https:\/\/www2.census.gov\/geo\/tiger\/TIGER2016\/FEATNAMES\//' -e 's/["'"'"']$//' | \
      xargs -I {} -P 24 -n 1 sh -c 'export f={}; curl -s -o ./featnames/$(basename $f) $f; echo $f'
   
   mkdir -p ./roads
   curl -s https://www2.census.gov/geo/tiger/TIGER2016/ROADS/ | \
      grep -o '<a href=['"'"'"][^"'"'"']*['"'"'"]' | \
      grep roads | \
      sed -e 's/^<a href=["'"'"']/https:\/\/www2.census.gov\/geo\/tiger\/TIGER2016\/ROADS\//' -e 's/["'"'"']$//' | \
      xargs -I {} -P 24 -n 1 sh -c 'export f={}; curl -s -o ./roads/$(basename $f) $f; echo $f'
   ```

2. Unzip the TIGER data into per-county directories.

   ```bash
   mkdir -p ./expanded
   find ./featnames -name '*.zip' -print | \
      xargs -t -L1 -P 24 -I {} /bin/sh -c 'export f={}; unzip -q $f -d ./expanded/$(basename $f _featnames.zip)'
   find /mnt/tiger/roads -name '*.zip' -print | \
      xargs -t -L1 -P 24 -I {} /bin/sh -c 'export f={}; unzip -q $f -d ./expanded/$(basename $f _roads.zip)'
   ```

3. Convert the `ROADS` Shapefiles and `FEATNAMES` DBF files into CSVs.

   ```bash
   find ./expanded -name '*_featnames.dbf' -print0 | \
      xargs -t -0 -P 24 -Ifile ogr2ogr -f CSV file.csv file
   find ./expanded -name '*_roads.shp' -print0 | \
      xargs -t -0 -P 24 -Ifile ogr2ogr -lco GEOMETRY=AS_WKT -f CSV file.csv file
   ```

4. Use the included Python script to join the `ROADS` and `FEATNAMES` data sets and expand the abbreviated road names. The resulting data will be written as newline-separated GeoJSON features.

   ```bash
   find ./expanded -name '*_roads.shp' -print | \
      xargs -t -L1 -P 24 -Ifile /bin/sh -c 'f=file; d=$(dirname $f); b=$(basename $f _roads.shp) && \
      python merge_tiger_roads.py $d/${b}_roads.shp.csv $d/${b}_featnames.dbf.csv $d/$b.expanded.json'
   ```

5. Run the resulting CSV through tippecanoe to generate an mbtiles file.

   ```bash
   (find ./expanded -type f -name '*.expanded.json' -exec cat {} \;) | \
     ./tippecanoe \
       --no-line-simplification \
       --buffer=0 \
       --read-parallel \
       --temporary-directory=./tmp \
       --base-zoom=12 \
       --maximum-zoom=12 \
       --minimum-zoom=12 \
       -o ./tiger_roads.mbtiles
   ```

6. Send the mbtiles file to MapBox for rendering.
