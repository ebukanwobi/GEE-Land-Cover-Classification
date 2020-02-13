# GEE-Land-Cover-Classification
Using GEE to estimate agricultural land in Brazil
  //Let's centre the map view over our ROI
Map.centerObject(initial, 7);
  //DEM scene 
var demelevation = dem.select('elevation');//elevation data
  //print(demelevation)
var slope = ee.Terrain.slope(demelevation);//slope of the landscape
  //print(slope)

  //sentinel2
  // Now select your image type!
var collection = ee.ImageCollection('COPERNICUS/S2_SR') // searches all sentinel 2 imagery pixels...
  .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 5)) // ...filters on the metadata for pixels less than 10% cloud
  .filterDate('2018-09-01' ,'2019-06-30') //... chooses only pixels between the dates you define here
  .filterBounds(initial) // ... that are within your aoi
//print(collection) // this generates a JSON list of the images (and their metadata) which the filters found in the right-hand window.
  
  /// so far this is finding all the images in the collection which meets the critera- the latest on top. To get a nice blended-looking mosaic, 
  // try some of the tools for 'reducing' these to one pixel (or bands of pixels in a layer stack). 


var medianpixels = collection.median() // This finds the median value of all the pixels which meet the criteria. 

var medianpixelsclipped = medianpixels.clip(initial) // this cuts up the result so that it fits neatly into your aoi
                                                                  // and divides so that values between 0 and 1      
var medianpixelsclippednew = medianpixels.clip(initial).divide(10000)

// Now visualise the mosaic as a natural colour image. 
//Map.addLayer(medianpixelsclippednew, {bands: ['B12', 'B11', 'B4'], min: 0, max: 1, gamma: 1.5}, 'Sentinel_2 mosaic',false)
//Map.addLayer(medianpixelsclippednew, {bands: ['B4', 'B3', 'B2'], min: 0, max: 1, gamma: 1.5}, 'Sentinel_2 truecolour',false)
//Map.addLayer(medianpixelsclippednew, {bands: ['B8', 'B4', 'B3'], min: 0, max: 1, gamma: 1.5}, 'Sentinel_2 falsecolour',false)

// Calculate NDVI
var image_ndvi = medianpixelsclipped.normalizedDifference(['B8','B4']);
//print(image_ndvi);

// Compute the Normalized Difference Vegetation Indices
var aerosol = medianpixelsclipped.select('B1');
var green = medianpixelsclipped.select('B3');
var red = medianpixelsclipped.select('B4');
var vre1 = medianpixelsclipped.select('B5');
var vre2 = medianpixelsclipped.select('B6');
var vre3 = medianpixelsclipped.select('B7');
var nir = medianpixelsclipped.select('B8');
var vre4 = medianpixelsclipped.select('B8A');
var swir1 = medianpixelsclipped.select('B11');

var brazilndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
var brazilndii = nir.subtract(swir1).divide(nir.add(swir1)).rename('NDII');
var brazilndwi = green.subtract(nir).divide(green.add(nir)).rename('NDWI');
var brazilsipi = nir.subtract(aerosol).divide(nir.subtract(red)).rename('SIPI'); //Calculate SIPI1
var brazilpssr = nir.divide(red).rename('PSSR'); //Calculate PSSRb1
var brazilsavi = (nir.subtract(red).divide(nir.add(red).add(0.428))).multiply(1+0.428).rename('SAVI');//Calculate SAVI

//print(brazilndvi);
var ndvi = brazilndvi.select('NDVI');

// Filter the collection for the VV product from the descending track
var collectionVV = ee.ImageCollection('COPERNICUS/S1_GRD')
    .filter(ee.Filter.eq('instrumentMode', 'IW'))
    .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
    .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
    .filterDate('2018-09-01' ,'2019-06-30')
    .filterBounds(initial)
    .select(['VV']);
//print(collectionVV);
// Filter the collection for the VH product from the descending track
var collectionVH = ee.ImageCollection('COPERNICUS/S1_GRD')
    .filter(ee.Filter.eq('instrumentMode', 'IW'))
    .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
    .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
    .filterDate('2018-09-01','2019-06-30')
    .filterBounds(initial)
    .select(['VH']);
//print(collectionVH);

var VV = collectionVV.median().clip(initial); //median of selected band
var VH = collectionVH.median().clip(initial);//median of selected band
var ratio = VV.divide(VH).rename('ratio'); //band ratio

//print(VV);
//print(VH);
//print(ratio);

var vvtoBytes = VV.toByte();
//print(vvtoBytes);

var vhtoBytes = VH.toByte();
//print(vhtoBytes);

var ratiotoBytes = ratio.toByte();
//print(ratiotoBytes);

// Adding the VV layer to the map
//Map.addLayer(VV, {min: -14, max: -7}, 'VV',false);
//Map.addLayer(VH, {min: -20, max: -7}, 'VH',false);

// Compute standard deviation (SD) as texture of the NDVI

var brazilb5sd = vre1.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilb6sd = vre2.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilb7sd = vre3.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilb8asd = vre4.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilndvitexture = brazilndvi.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilndvimean = brazilndvi.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilndvivariance = brazilndvi.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilndiimean = brazilndii.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilndwimean = brazilndwi.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilsipimean = brazilsipi.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilsavimean = brazilsavi.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilndiisd = brazilndii.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilndwisd = brazilndwi.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilsipisd = brazilsipi.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilsavisd = brazilsavi.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilndiivar = brazilndii.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilndwivar = brazilndwi.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilsipivar = brazilsipi.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilsavivar = brazilsavi.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilVVtexture = VV.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilVVmean = VV.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilVVvariance = VV.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilVHtexture = VH.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilVHmean = VH.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilVHvariance = VH.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

var brazilratiomean = ratio.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(5),
});

var brazilratiosd = ratio.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(5),
});

var brazilratiovariance = ratio.reduceNeighborhood({
  reducer: ee.Reducer.variance(),
  kernel: ee.Kernel.circle(5),
});

//Layer Stack
var mergedCollection = medianpixelsclipped.addBands(VV).addBands(VH).addBands(ratio).addBands(brazilndvi).addBands(brazilndii).addBands(brazilndwi).addBands(brazilsipi).addBands(brazilpssr).addBands(brazilsavi).addBands(demelevation).addBands(slope).addBands(brazilndvimean).addBands(brazilndwimean).addBands(brazilndiimean).addBands(brazilndvivariance).addBands(brazilndvitexture).addBands(brazilndiimean).addBands(brazilndwimean).addBands(brazilsavimean).addBands(brazilsipimean).addBands(brazilndiisd).addBands(brazilndwisd).addBands(brazilsavisd).addBands(brazilsipisd).addBands(brazilndiivar).addBands(brazilndwivar).addBands(brazilsavivar).addBands(brazilsipivar).addBands(brazilVVtexture).addBands(brazilVVmean).addBands(brazilVVvariance).addBands(brazilVHtexture).addBands(brazilVHmean).addBands(brazilVHvariance).addBands(brazilratiomean).addBands(brazilratiosd).addBands(brazilratiovariance).addBands(brazilb5sd).addBands(brazilb6sd).addBands(brazilb7sd).addBands(brazilb8asd);
//print('mergedCollection: ', mergedCollection);


var bands = ['B1','B2','B3','B4', 'B5','B6','B7','B8','B8A','B9','B11','B12','VV','VH'];
var bandvi = ['nd' ,'nd_1','nd_2','nd_3','sipi','pssr','savi'];
var band= ['slope','B2','B3','B4', 'B5','B6','B7','B8','B8A','VV','VH','ratio','NDVI','SIPI','NDII','NDWI','SAVI','NDVI_stdDev','NDII_stdDev','NDWI_stdDev','SAVI_stdDev','SIPI_stdDev','VV_stdDev','VH_stdDev','ratio_stdDev','B5_stdDev','B6_stdDev','B7_stdDev','B8A_stdDev'];

//training data
//var classNames = urbantrain.merge(watertrain).merge(foresttrainagain).merge(coffeetrainagain).merge(opengroundtrained).merge(morenoncoffeetrain)
//var classNames = urbantrain.merge(watertrain).merge(foresttrainagain).merge(coffeetrainagain).merge(opengroundtrained).merge(morenoncoffeetrain).merge(trnewpolycoffee).merge(trpolynoncoffeee).merge(trpolyurban).merge(trpolyforestt).merge(trpolyopenground).merge(trpolywaterr);
var classNames = trnewpolycoffee.merge(newcoffeepolytr).merge(coffeepolystr).merge(coffeenewtrain).merge(newcoffeetrnew).merge(trnewpolyforest).merge(newforesttrnew).merge(newbeloforest).merge(geeforest).merge(trpolyurban).merge(newbelourban).merge(watertrain).merge(trnewpolywateragain).merge(newbelowater).merge(trnewpolynoncoffee).merge(newnoncoffeetrneww).merge(newbelononcoffee).merge(trpolyopenground).merge(foresttrainagain).merge(coffeetrainagain).merge(opengroundtrained).merge(morenoncoffeetrain).merge(shrubtr).merge(newopengroundtr).merge(newnoncoffeetr).merge(roadtr).merge(coffeepttr).merge(forestpttr).merge(geenoncoffee).merge(noncoffeepttr).merge(polyroadtr).merge(shrubpolytr).merge(waterpttr).merge(newpolyroadtr).merge(newbeloroad).merge(newbeloopenground);
//var classNames = urbantrain.merge(watertrain).merge(foresttrainagain).merge(coffeetrainagain).merge(opengroundtrained).merge(morenoncoffeetrain).merge(trpolycoffee).merge(trpolynoncoffee).merge(trpolyurban).merge(trpolyforest).merge(trpolyopenground);
var testnames = urbantested.merge(newwatertest).merge(foresttested).merge(coffeetested).merge(opengroundtested).merge(noncoffeetested);

//print(classNames);
var label = 'descriptio'

//testing data
//var testData = TrainingData.filter(ee.Filter.gt("random",0.8))
//var TestSample = img.sampleRegions(testData,["land_class"],30);

//Create training data
var training = mergedCollection.select(band).sampleRegions({
  collection: classNames,
  properties: [label],
  scale: 15
});

//train classifier
var classifier = ee.Classifier.randomForest({
  numberOfTrees: 30,
  variablesPerSplit: 3
}).train(training,label,band);

//Run the classification
var classified = mergedCollection.select(band).classify(classifier);

//Display classification
Map.addLayer(classified.clip(initial),{min: 0, max: 5, palette: ['purple', 'green', 'red','blue','yellow','black']},'classification');

var confMatrix = classifier.confusionMatrix()
print('Resubstitution error matrix: ', confMatrix);
print('Training overall accuracy: ', confMatrix.accuracy());

//Create testing data
var validation = classified.sampleRegions({
  collection: testnames,
  properties: ['descriptio'],
  scale: 15
});

//Compare the landcover of your validation data against the classification result
var testAccuracy = validation.errorMatrix('descriptio', 'classification');
//Print the error matrix to the console
print('Validation error matrix: ', testAccuracy);
//Print the overall accuracy to the console
print('Validation overall accuracy: ', testAccuracy.accuracy());


// Export the image, specifying scale and region.
Export.image.toDrive({
  image: classified,
  description: 'RFLandCoverClassificationBrazilNW',
  scale: 20,
  maxPixels:1e13,
  region: northwest,
  });
  
  // Export the image, specifying scale and region.
Export.image.toDrive({
  image: classified,
  description: 'RFLandCoverClassificationBrazilNE',
  scale: 20,
  maxPixels:1e13,
  region: northeast,
  });
  
  // Export the image, specifying scale and region.
Export.image.toDrive({
  image: classified,
  description: 'RFLandCoverClassificationBrazilSW',
  scale: 20,
  maxPixels:1e13,
  region: southwest,
  });
  
  // Export the image, specifying scale and region.
Export.image.toDrive({
  image: classified,
  description: 'RFLandCoverClassificationBrazilSE',
  scale: 20,
  maxPixels:1e13,
  region: southeast,
  });
  
    // Export the image, specifying scale and region.
Export.image.toDrive({
  image: classified,
  description: 'RFLandCoverClassificationBrazil%accuracy',
  scale: 20,
  maxPixels:1e13,
  region: roibrazil,
  });
