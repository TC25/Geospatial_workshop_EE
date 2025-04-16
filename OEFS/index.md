

# Vegetation Change Analysis: From Local to Global

In this tutorial, we will walk you through a global to local analysis of vegetation change using Google Earth Engine (GEE).

## Introduction

Understanding vegetation change over time helps in monitoring ecological health, land use planning, and managing natural resources. This tutorial will introduce you to the steps required to analyze and visualize vegetation change using satellite data in GEE. A focus will be on exploring how easy it is scale your workflow to perform planetary-scale analysis using the platform.

## Local Analysis: Chicago

This section covers the analysis of vegetation change using the Normalized Difference Vegetation Index (NDVI) in Chicago.

### Loading Data

First, load the feature collection of Chicago's census tracts from user assets and add a water mask.

```javascript

//*******This script should be run on the Google Earth Engine Javascript API (https://earthengine.google.com/)*********//

var ROI = ee.FeatureCollection('users/tirthankar25/chicago')
var Water = ee.Image("JRC/GSW1_0/GlobalSurfaceWater").select("occurrence");

print(ROI)
Map.addLayer(ROI, {}, "Chicago Shapefile") 
Map.setCenter(-87.679, 41.847, 10)
```

### Working with MODIS Data
#### Load the MODIS image collection and filter it for NDVI data.

```javascript
var MODIS = ee.ImageCollection('MODIS/006/MOD13A1');
print(MODIS)

var SumFilter = ee.Filter.dayOfYear(152, 273);
var NDVI = MODIS.select('NDVI')
var NDVI_dateofint = NDVI.filterDate('2014-01-01', '2018-12-30').filter(SumFilter);

var NDVI_mean = NDVI_dateofint.mean()
var NDVI_final = NDVI_mean.multiply(.0001)
var NDVI_Chicago = NDVI_final.clip(ROI).updateMask(Water.mask().not())
```

### Reducing Data by Region
#### Calculate spatial mean NDVI values for each census tract.

```javascript
function Neighborhood_mean(feature) {
  var reduced_NDVI = NDVI_Chicago.reduceRegion({ reducer: ee.Reducer.mean(), geometry: feature.geometry(), scale: 500 })
  return feature.set({ 'NDVI': reduced_NDVI.get('NDVI') })
}

var Reduced = ROI.map(Neighborhood_mean)
print(Reduced)

var NDVI_MODIS = Reduced.reduceToImage({ properties: ee.List(['NDVI']), reducer: ee.Reducer.first() })
print(NDVI_MODIS)
```

### Export Data
#### Export the data as images or tables for further analysis.

```javascript
Export.image.toDrive({ image: NDVI_MODIS, description: 'Chicago_NDVI', folder: 'Workshop', region: ROI.geometry().bounds(), scale: 500 })
Export.table.toDrive({ collection: Reduced, description: 'Chicago_censustracts', folder: 'Workshop', fileFormat: 'CSV' })
```

## Global Analysis: Tree Cover Change
### Now, let's analyze global tree cover change between 2001 and 2015.
### Loading Data
#### Load the MODIS tree cover data for the years 2001 and 2015.

```javascript
var MODIS = ee.ImageCollection('MODIS/006/MOD44B');
var TREE = MODIS.select('Percent_Tree_Cover')

var TREE_2001 = TREE.filterDate('2001-01-01', '2001-12-31').mean()
var TREE_2015 = TREE.filterDate('2015-01-01', '2015-12-31').mean()
```

### Calculating Change
#### Compute the change in tree cover between 2001 and 2015.

```javascript
var TREE_change = TREE_2015.subtract(TREE_2001);

var TREE_increase = TREE_change.remap({ from: ee.List([-999]), to: ee.List([-999]), defaultValue: 1 }).mask(TREE_change.gt(0))
var TREE_same = TREE_change.remap({ from: ee.List([-999]), to: ee.List([-999]), defaultValue: 0 }).mask(TREE_change.eq(0))
var TREE_decrease = TREE_change.remap({ from: ee.List([-999]), to: ee.List([-999]), defaultValue: -1 }).mask(TREE_change.lt(0))

var TREE_FINAL = ee.ImageCollection([TREE_increase.float(), TREE_same.float(), TREE_decrease.float()]).mosaic()
```

### Exporting Data
#### Export the final tree cover change map.

```javascript
var ROI = ee.Geometry.Rectangle(-180, -90, 180, 90);
var ROI = ee.Geometry(ROI, null, false);

Export.image.toDrive({ image: TREE_FINAL, description: 'TREE_FINAL_image', folder: 'OEFS', region: ROI, scale: 250 })
```

Now you have a complete tutorial on analyzing vegetation change from a local (Chicago) to a global scale using the Google Earth Engine. Happy analyzing!
