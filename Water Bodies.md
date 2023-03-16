## Identify Water Bodies

We are going now to identify the water bodies using a Sentinel 2 image.

1. Using the same dates that you used on the [NDVI step](/GetNDVI.md) obtain the water raster for the city.

Remember to have your study area vector as an asset on your Google Earth Engine Account.

```javascript
var City = ee.FeatureCollection("path to your asset");

/**
 * Function to mask clouds using the Sentinel-2 QA band
 * @param {ee.Image} image Sentinel-2 image
 * @return {ee.Image} cloud masked Sentinel-2 image
 */
function maskS2clouds(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}

var dataset = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filterBounds(City)
                  //.filter(ee.Filter.contains('.geo', geometry))
                  .filterDate('2019-07-01', '2019-08-31')
                  // Pre-filter to get less cloudy granules.
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',10))
                  .map(maskS2clouds)
                  //.sort('CLOUD_COVER')
                  //.first();
                  

print (dataset)
var median = dataset.median().clip(City)

var nir = median.select('B8');
var red = median.select('B4');

var MuWIR = median.expression('(-4*ND23)+(2*ND38)+(2*(ND312)-ND311)',{
  'ND23':median.normalizedDifference(['B2' , 'B3']),
  'ND38':median.normalizedDifference(['B3' , 'B8']),
  'ND312':median.normalizedDifference(['B3' , 'B12']),
  'ND311':median.normalizedDifference(['B3' , 'B11'])
  
});

var projection = median.select('B2').projection().getInfo()
print(projection)

Map.centerObject(City, 12);

var ndviParams = {min: -1, max: 1, palette: ['Darkgreen', 'white', 'Darkblue']};
Map.addLayer(MuWIR, ndviParams, "MuWIR layer");


Export.image.toDrive({
  image: MuWIR,
  description: 'MuWIR',
  crs: 'EPSG:28992',
  scale: 10,
  region: City,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});

```

This code uses a Water index method from Wang et al., 2018

[Multi-Spectral Water Index (MuWI): A Native 10-m Multi-Spectral Water Index for Accurate Water Mapping on Sentinel-2](https://doi.org/10.3390/rs10101643)

2. Download your raster file and open it in your GIS software (QGIS, ArcGIS Pro, etc.)
3. Identify the threshold for water in your own layer. For the city of Enschede the threshold was set at 0.3.
4. Use the Reclassify tool for updating your raster values to Water (1) and No Water (0). It should look something like this:

|Value | New Value|
|------|----------|
|>= 0.3|     1    |
|< 0.3 |     0    |

5. Run the Raster to Polygon tool, save this layer as WaterBodies.
6. On your polygon Layer, select those polygons with value of 0. You can do this using the select by attributes or select by expression tool. Your SQL Command should look like:

```SQL
SELECT * WHERE gridcode = 0
```
7. Now delete all those polygons and save your edits.
8. Some white roofs might show up as water bodies, to clean your data we are going to use the  [Building Footprint](/Building Footprint.md).
9. Use the Erase tool to elmiminate all the polygons that are intersecting a Building, that way we will have a clean water Bodies layer detected at a 10m Resolution





