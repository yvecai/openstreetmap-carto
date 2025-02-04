# Installation

This document describes how to manually configure your system for running OpenStreetMap Carto. If you prefer quick, platform independent setup for a development environment, without the need to install and configure tools by hand, follow a Docker installation guide in [DOCKER.md](DOCKER.md).

## OpenStreetMap data
You need OpenStreetMap data loaded into a PostGIS database (see below for [dependencies](#dependencies)). These stylesheets expect a database generated with osm2pgsql using the pgsql backend (table names of `planet_osm_point`, etc), the default database name (`gis`), and the [lua transforms](https://osm2pgsql.org/doc/manual.html#lua-tag-transformations) documented in the instructions below.

Start by creating a database

```
sudo -u postgres createuser -s $USER
createdb gis
```

Enable PostGIS and hstore extensions with

```
psql -d gis -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```

then grab some OSM data. It's probably easiest to grab an PBF of OSM data from [Geofabrik](https://download.geofabrik.de/). Once you've done that, import with osm2pgsql:

```
osm2pgsql -G --hstore --style openstreetmap-carto.style --tag-transform-script openstreetmap-carto.lua -d gis ~/path/to/data.osm.pbf
```

You can find a more detailed guide to setting up a database and loading data with osm2pgsql at [switch2osm.org](https://switch2osm.org/serving-tiles/manually-building-a-tile-server-16-04-2-lts/).

### Disable JIT

We do not recommend [PostgreSQL JIT](https://www.postgresql.org/docs/current/jit-reason.html), which is on by default in PostgreSQL 12 and higher. JIT is benifitial for slow queries where executing the SQL takes substantial time and that time is not spent in function calls. This is not the case for rendering, where most time is spent either fetching from disk, in PostGIS functions, or the query is fast. In theory, the query planner will only use JIT on slower queries, but it is known to get the type of queries map rendering requries wrong.

Disabling JIT is **essential** for use with Kosmtik and other style development tools.

JIT can be disabled with `psql -d gis -c 'ALTER SYSTEM SET jit=off;' -c 'SELECT pg_reload_conf();'` or any other means of adjusting the PostgreSQL config.

### Custom indexes
Custom indexes are required for rendering performance and are essential on full planet databases.

```
psql -d gis -f indexes.sql
```

## Scripted download
Some features are rendered using preprocessed shapefiles.

To download them and import them into the database you can run the following script

```
scripts/get-external-data.py
```

The script downloads shapefiles, loads them into the database and sets up the tables for rendering. Additional script option documentation can be seen with `scripts/get-external-data.py --help`.

## Fonts
The stylesheet uses Noto, an openly licensed font family from Google with support for multiple scripts. The stylesheet uses Noto's "Sans" style where available. If not available, this stylesheet uses another appropriate style of the Noto family. The "UI" version is used where available, with its vertical metrics which fit better with Latin text.

Hanazono is used a fallback for seldom used CJK characters that are not covered by Noto.

For more details, see the documentation at [fonts.mss](style/fonts.mss).

To download the fonts, run the following script

```
scripts/get-fonts.sh
```

## Dependencies

For development, a style design studio is needed.
* [Kosmtik](https://github.com/kosmtik/kosmtik) - Kosmtik can be launched with `node index.js serve path/to/openstreetmap-carto/project.mml` The 0.0.17 release of Kosmtik is not enough because we need up-to-date CartoCSS and Mapnik versions. To install the current master branch of Kosmtik, you can clone the Kosmtik repository and execute `npm install` within it.

[TileMill](https://tilemill-project.github.io/tilemill/) is not officially supported, but you may be able to use a recent TileMill version by copying or symlinking the project directly into your Mapbox/project directory.

To display any map a database containing OpenStreetMap data and some utilities are required

* [PostgreSQL](https://www.postgresql.org/)
* [PostGIS](https://postgis.net/)
* [osm2pgsql](https://github.com/openstreetmap/osm2pgsql#installing) to [import your data](https://switch2osm.org/serving-tiles/updating-as-people-edit/) into a PostGIS database
* Python 3 with the psycopg2, yaml, and requests libraries (`python3-psycopg2`, `python3-yaml`, `python3-requests` packages on Debian-derived systems)
* `ogr2ogr` for loading shapefiles into the database (`gdal-bin` on Debian-derived systems)

### Optional development dependencies

Some colours, SVGs and other files are generated with helper scripts. Not all users will need these dependencies

* Python and Ruby to run helper scripts
* [Color Math](https://github.com/gtaylor/python-colormath) and [numpy](https://numpy.org/) if running generate_road_colors.py helper script (may be obtained with `pip install colormath numpy`)

### Additional deployment dependencies

For deployment, CartoCSS and Mapnik are required.

* [CartoCSS](https://github.com/mapbox/carto) >= 1.2.0 *(we're using YAML)*
* [Mapnik](https://github.com/mapnik/mapnik/wiki/Mapnik-Installation) >= 3.0.22

With CartoCSS you compile these sources into a Mapnik compatible XML file. When running CartoCSS, specify the Mapnik API version you are using (at least 3.0.22: `carto -a "3.0.22"`).