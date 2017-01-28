# D3 Demo - Population Visualization of Texas

Following along with Mike Bostock's 4 part blog series [Command-Line Cartography](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c#.ff2cxnmc9).

## Dependencies
- [shapefile](https://github.com/mbostock/shapefile)
- [D3 Geo Projection](https://github.com/d3/d3-geo-projection)
- [ndjson-cli](https://github.com/mbostock/ndjson-cli)
- [D3](https://github.com/d3/d3)
- [TopoJSON](https://github.com/topojson/topojson/blob/master/README.md#api-reference)
```bash
npm install -g shapefile
npm install -g d3-geo-projection
npm install -g ndjson-cli
npm install -g d3
npm install -g topojson
```

## Data Source

### Geometry Data
Geometry Data as cartographic boundary files are curled from [US Census Bureau](http://www2.census.gov/geo/tiger/GENZ2014/shp/) for the State of Texas using a [FIPS code](https://en.wikipedia.org/wiki/Federal_Information_Processing_Standard_state_code) of 48.

``` bash
curl 'http://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_48_tract_500k.zip' -o cb_2014_48_tract_500k.zip
unzip -o cb_2014_48_tract_500k.zip
```
### Population Data
Population data was extracted using the US Census API's 2014  ([request API Key](http://api.census.gov/data/key_signup.html)). The request below is for Total Population data — See 'B01003_001E' in [ACS API variables](http://api.census.gov/data/2014/acs5/variables.html) — from the 2014 American Community Survey API Endpoint ([see docs](http://api.census.gov/data/2014/acs5/examples.html)).

```bash
curl "http://api.census.gov/data/2014/acs5?get=B01003_001E&for=tract:*&in=state:48&key=${CENSUSAPIKEY}" -o cb_2014_48_tract_B01003.json
```
Note: I created a .env file for storing my Census API Key as an environment variable.

#Part 1
## Data Manipulation
Convert shapefile to GeoJson using [Mike Bostock's shapefile parser](https://github.com/mbostock/shapefile) with a command-line interface, [shp2json](https://github.com/mbostock/shapefile/blob/master/README.md#shp2json).
```bash
shp2json cb_2014_48_tract_500k.shp -o tx.json
```

## Quick SVG Visualization
First, I needed to find an appropriate projection for Texas. I chose NAD83 / Texas North (EPSG:32137) from [D3 Stateplane] (https://github.com/veltman/d3-stateplane).
``` bash
    geoproject 'd3.geoConicConformal().parallels([34 + 39 / 60, 36 + 11 / 60]).rotate([101 + 30 / 60, -34]).fitSize([960,960], d)' < tx.json > tx-north.json
```
### Preview Visualization
Convert projection into SVG

``` bash
geo2svg -w 960 -h 960 < tx-north.json > tx-north.svg
```
# Part II

## Create Feature Stream of GeoJSON
GeoJSON is a huge collection of features which we need to access a feature at a time — enter [ndjson-split](https://github.com/mbostock/ndjson-cli/blob/master/README.md#split):
```bash
ndjson-split 'd.features' \
  < tx-north.json \
  > tx-north.ndjson
```
## Generate Feature ID for Boundaries
The following command grabs the GEOID from a feature's property, removes the FIPs code assigns it as the features ID.
```bash
ndjson-map 'd.id = d.properties.GEOID.slice(2), d' \
  < tx-north.ndjson \
  > tx-north-id.ndjson
```
## Convert US Census Population Data to NDJSON
The US Census population file is a JSON array. To convert it to an NDJSON stream, use ndjson-cat (to remove the newlines), ndjson-split (to separate the array into multiple lines) and ndjson-map (to reformat each line as an object). You can run these individually, but here’s how to do it all in one go ([Source](https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3#.ccsy9stcw)).

It is important to note that the ID we are using for our boundaries is simply the county ID + the tract ID. Remember, we removed the state identifier from the boundary GEOID.

```bash
ndjson-cat cb_2014_48_tract_B01003.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[2] + d[3], B01003: +d[0]}' \
  > ../cb_2014_48_tract_B01003.ndjson
```
Note: I output the NDJSON in my parent directory (../)

## Join population data with geometry data
Join the two data sets using the ID we added to our cartographic boundaries.

```bash
ndjson-join 'd.id' \
  tx-north-id.ndjson \
  cb_2014_48_tract_B01003.ndjson \
  > tx-north-join.ndjson
```

## Calculate Population Density (people/mi^2)
Calculate population density by dividing the boundary population (d[1].B01003) by the land area (d[0].properties.ALAND). Don't forget we need to convert our units from meters squared to miles squared.

``` bash
ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < tx-north-join.ndjson \
  > tx-north-density.ndjson
```
Note from the Author: Note that the density value is floored rather than rounded. We don’t need the extra precision, so either results in a smaller output file. But since we will later apply a threshold color encoding in the choropleth, rounding would be inappropriate: for example, it would change an effective threshold of 4,000 to 3,999.5! ([Source](https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3#.ccsy9stcw))

## Convert back to GeoJSON

``` bash
ndjson-reduce 'p.features.push(d), p' '{type: "FeatureCollection", features: []}' \
  < tx-north-density.ndjson \
  > tx-north-density.json
```
## Generate SVG choropleth

### Define fill property using D3
Next use [ndjson-map](https://github.com/mbostock/ndjson-cli/blob/master/README.md#map), requiring D3 via -r d3, and defining a fill property using a [sequential scale](https://github.com/d3/d3-scale/blob/master/README.md#sequential-scales) with the [Viridis](https://github.com/d3/d3-scale/blob/master/README.md#interpolateViridis) color scheme:

```bash
ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)' \
  < tx-north-density.ndjson \
  > tx-north-color.ndjson
```

### Create SVG from GeoJSON
```bash
geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < tx-north-color.ndjson \
  > tx-north-color.svg
```
# Part III

Next we will 1.) Simplify 2.) Quantize and 3.) Compress our GeoJSON.

I highly recommend reading [Part III of Mike Bostock's Blog post](https://medium.com/@mbostock/command-line-cartography-part-3-1158e4c55a1e#.4o4lpsif0).

## Convert GeoJSON to TopoJson
```bash
geo2topo -n \
  tracts=tx-north-density.ndjson \
  > tx-north-topo.json
```

## Further reduce TopoJson
Use [toposimplify](https://github.com/topojson/topojson-simplify/blob/master/README.md#toposimplify) to further shrink our file.
```bash
toposimplify -p 1 -f \
  < tx-north-topo.json \
  > tx-simple-topo.json
```
## Topoquantize
Reduce floats to integers

```bash
topoquantize 1e5 \
  < tx-simple-topo.json \
  > tx-quantized-topo.json
```

## Merge tracts within county
Since census tracts compose hierarchically into counties, we can derive county geometry using topomerge!

```bash
topomerge -k 'd.id.slice(0, 3)' counties=tracts \
  < tx-quantized-topo.json \
  > tx-merge-topo.json
```

## Remove exterior county borders

``` bash
topomerge --mesh -f 'a !== b' counties=counties \
  < tx-merge-topo.json \
  > tx-topo.json
  ```
# Part IV
