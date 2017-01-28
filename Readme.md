# D3 Demo - Population Visualization of Texas

Following along with Mike Bostock's 4 part blog series [Command-Line Cartography](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c#.ff2cxnmc9).

## Data Source

Data is curled from [US Census Bureau](http://www2.census.gov/geo/tiger/GENZ2014/shp/) for the State of Texas.

``` bash
curl 'http://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_48_tract_500k.zip' -o cb_2014_48_tract_500k.zip
unzip -o cb_2014_48_tract_500k.zip
```

## Dependencies
- [shapefile](https://github.com/mbostock/shapefile)
- [D3 Geo Projection](https://github.com/d3/d3-geo-projection)
- [ndjson-cli](https://github.com/mbostock/ndjson-cli)
```bash
npm install -g shapefile
npm install -g d3-geo-projection
npm install -g ndjson-cli
```


## Data Manipulation
Convert shapefile to GeoJson
```bash
shp2json cb_2014_48_tract_500k.shp -o tx.json
```

## Quick Visualization
First, I needed to find an appropriate projection for Texas. I chose NAD83 / Texas North (EPSG:32137) from [D3 Stateplane] (https://github.com/veltman/d3-stateplane).
``` bash
    geoproject 'd3.geoConicConformal().parallels([34 + 39 / 60, 36 + 11 / 60]).rotate([101 + 30 / 60, -34]).fitSize([960,960], d)' < tx.json > tx-north.json
```
### Preview Visualization
Convert projection into SVG

``` bash
geo2svg -w 960 -h 960 < tx-north.json > tx-north.svg
```
