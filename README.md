# Mapping US Congressional Districts

Follow the steps below to create a web map of United States congressional districts from Census Bureau data using Mapbox. You can also use this to create a lat/lng-to-congressional district API.

These instructions create maps for the 116th Congress using district boundaries as of the 2018 elections. (These differed from the 115th Congress boundaries because of court-ordered redistricting in Pennsylvania in 2018.)

You will need an account on Mapbox.com. Then follow the commands below from the Mac OS X or Ubuntu terminal.

Why use [Tippecanoe](https://github.com/mapbox/tippecanoe)? Using Tippecanoe provides more control over how the geometries are tiled into a map. For comparison, using the Mapbox Studio default upload will not show a zoomed-out full country view of the data because the boundaries are so detailed; the default upload thinks you are only interested in looking closer at the data. Tippecanoe stops  oversimplification of the geometry and also specifies a min/max zoom level.

#### Dependencies:

On OS X, install required dependencies with Homebrew:

```
brew install tippecanoe gdal node
```

On Ubuntu, you'll need node:

```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash
nvm install 5.0
```

and gdal and Tippecanoe, which must be built from sources:

```
sudo apt-get install gdal-bin libprotobuf-dev protobuf-compiler libsqlite3-dev
git clone https://github.com/mapbox/tippecanoe
cd tippecanoe
make
cd ..
```

#### Setup:

Download this repository and then use `npm` to install a few more dependencies:

```
git clone https://github.com/govtrack/congress-maps.git
cd congress-maps
npm install
```

#### Creating the map:

To complete these steps, run the commands below. Set `MAPBOX_USERNAME` to your Mapbox username, `MAPBOX_DEFAULT_ACCESS_TOKEN` to your Mapbox default public token, and `MAPBOX_ACESS_TOKEN` to a `uploads:write` scope access token from your [Mapbox account](https://www.mapbox.com/studio/account/tokens).

```
# create directory to store data
mkdir data

# dowload Census boundaries data, unzip the data, and convert it to GeoJSON
wget -P data ftp://ftp2.census.gov/geo/tiger/TIGER2018/CD/tl_2018_us_cd116.zip
unzip data/tl_2018_us_cd116.zip -d ./data/
ogr2ogr -f GeoJSON -t_srs crs:84 data/congressional_districts.geojson data/tl_2018_us_cd116.shp

# download new 2020 North Carolina CDs
# (h/t https://blogs.sas.com/content/graphicallyspeaking/2019/11/25/plotting-ncs-new-congressional-districts-maps-for-2020/)
wget -O data/NC_HB1029_3.zip https://webservices.ncleg.gov/ViewBillDocument/2019/6953/0/HB%201029,%203rd%20Edition%20-%20Shapefile
unzip data/NC_HB1029_3.zip -d ./data/
ogr2ogr -f GeoJSON -t_srs crs:84 data/congressional_districts_nc.geojson data/C-Goodwin-A-1-TC.shp

# run processing on data
node process.js

# create Mapbox vector tiles from data
tippecanoe/tippecanoe -o data/cd-117-2020.mbtiles -f -Z 0 -z 12 -B0 -pS -n "117th Congress (2020 Election) Congressional Districts" data/map.geojson

# setup Mapbox account name, default public token, and write-scoped token
export MAPBOX_USERNAME=<your mapbox username>
export MAPBOX_DEFAULT_ACCESS_TOKEN=<your mapbox default access token>
export MAPBOX_WRITE_SCOPE_ACCESS_TOKEN=<your mapbox write scope access token>

# upload map data to Mapbox.com
node upload.js data/cd-117-2020.mbtiles "cd-117-2020" "US_Congressional_Districts_117th_Congress_2020_Elections"
```

Check out [mapbox.com/studio](https://www.mapbox.com/studio) to see updates on data processing. Once Mapbox is finished processing your upload, you can make a map style. Click New Style. Choose a template --- I last tried Navigation - Day, which seemed nice. Then create layers:

* Create a new layer. In Select Data, Select the uploaded tileset as the source. Change the type to fill. In Style, click Style With Data Conditions. Select color_index as the data value. Select 0, choose a color, click Done. Then click Add Another Condition and repeat for 1 through 4. Name the layer CD-Fills. Drag the layer all the way to the bottom of the layers but above "Land & water, land" so that it covers the base color for land but is behind all of the other map features. Click Style Across Zoom Range. Click the stop point for zoom 22 and set the opacity for each color to 0 and the zoom level to 15, so that the fills fade out when zoomed into street level.
* Create another layer with the same data named CD-Outlines. Set its type to Line. Click Width, then Style Across Zoom Range. Set the first zoom stop to zoom level 3 and line width 1px. Set the second zoom stop to zoom level 15 and the line width to 4px. Keep this layer at the top.
* Create another layer named CD-Labels. Set its type to Symbol. Click Text Field, style across zoom range. At zoom stop 0, set the text field to the data field title_short. Set the second stop to zoom level 7 and the text field to the data field title_long. I set the font to Montserrat Bold 17px.

I turned off country-label and state-label visibility.

#### Usage:

Use the files in the `example` directory as the basis for making a web map with functionality to focus on specific states or districts. To use this example web map, you'll need to edit `index.html` and insert your default public access token and the style URL and tileset ID from Mapbox.

After following the steps above, `index.html` will be a full page web map of U.S. Congressional districts. Host this file and the two supporting scripts (`states.js`, `bboxes.js`) on your website. If you don't want the interactive menu on your map, search through `index.html` and remove all sections of code that immediately follow the `INTERACTIVE MENU` line comment labels.

With this web map, you can show specific congressional districts using the URL hash. Set the location hash to `state={state abbreviation}` to show a specific state and add `&district={district number}` to specify a district within the state. The hash expects US Census two letter state abbreviations and district number conventions. At Large districts are numbered `00` and all other districts are two character numbers: `district=01`, `district=02`, ..., `district=15`, etc.

See the click handler for an example of how to use the Mapbox API to get the congressional district at a particular lat/lng coordinate.

#### Examples:

To show districts in the state of Virginia: http://www.aarondennis.org/congress-maps/example/#state=VA

To show the 5th district of California: http://www.aarondennis.org/congress-maps/example/#state=CA&district=05
