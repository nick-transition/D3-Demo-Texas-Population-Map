# D3 Demo - Population Visualization of Texas

Following along with Mike Bostock's 4 part blog series [Command-Line Cartography](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c#.ff2cxnmc9).

## Data Source

Data is curled from [US Census Bureau](http://www2.census.gov/geo/tiger/GENZ2014/shp/) for the State of Texas.

``` bash
curl 'http://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_48_tract_500k.zip' -o cb_2014_06_tract_500k.zip
unzip -o cb_2014_48_tract_500k.zip
```

## Dependencies
- [shapefile](https://github.com/mbostock/shapefile)
- [D3 Geo Projection](https://github.com/d3/d3-geo-projection)
```bash
npm install -g shapefile
npm install -g d3-geo-projection
```


## Data Manipulation
Convert shapefile to GeoJson
```bash
shp2json cb_2014_48_tract_500k.shp -o tx.json
```
