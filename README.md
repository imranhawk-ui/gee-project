# gee-project
// ============================================================
// KOLKATA SUMMER 2026 SUHI ANALYSIS
// LANDSAT 9 + DYNAMIC WORLD
// ============================================================


// ============================================================
// 1. STUDY AREA
// ============================================================

var cityPoint = ee.Geometry.Point([88.357, 22.5772]);

// 60 km study area around Kolkata
var studyArea = cityPoint.buffer(60000);

Map.centerObject(cityPoint, 10);


// ============================================================
// 2. LANDSAT 9 CLOUD / SHADOW MASK
// ============================================================

var maskLandsat9 = function(image) {

  var qa = image.select('QA_PIXEL');

  var mask = qa.bitwiseAnd(1 << 1).eq(0)   // Dilated cloud
    .and(qa.bitwiseAnd(1 << 2).eq(0))      // Cirrus
    .and(qa.bitwiseAnd(1 << 3).eq(0))      // Cloud
    .and(qa.bitwiseAnd(1 << 4).eq(0))      // Cloud shadow
    .and(qa.bitwiseAnd(1 << 5).eq(0));     // Snow

  return image.updateMask(mask);
};


// ============================================================
// 3. LANDSAT 9 SUMMER 2026
// ============================================================

var landsat9 = ee.ImageCollection(
  'LANDSAT/LC09/C02/T1_L2'
)
.filterBounds(studyArea)
.filterDate('2026-04-01', '2026-07-01')
.filter(ee.Filter.lt('CLOUD_COVER', 20))
.map(maskLandsat9);

print(
  'Number of Landsat 9 images:',
  landsat9.size()
);


// ============================================================
// 4. CONVERT ST_B10 TO LST
// ============================================================

var addLST = function(image) {

  var lstKelvin = image
    .select('ST_B10')
    .multiply(0.00341802)
    .add(149.0);

  var lstCelsius = lstKelvin
    .subtract(273.15)
    .rename('LST');

  return image.addBands(lstCelsius);
};


var processedCollection = landsat9.map(addLST);


// ============================================================
// 5. SUMMER LST COMPOSITE
// ============================================================

var summerLST = processedCollection
  .select('LST')
  .median()
  .clip(studyArea);

print(
  'Summer 2026 LST:',
  summerLST
);


// ============================================================
// 6. DISPLAY LST
// ============================================================

var lstVis = {
  min: 25,
  max: 50,
  palette: [
    'blue',
    'cyan',
    'green',
    'yellow',
    'orange',
    'red'
  ]
};

Map.addLayer(
  summerLST,
  lstVis,
  'Summer 2026 LST (°C)'
);


// ============================================================
// 7. DYNAMIC WORLD LAND COVER
// ============================================================
//
// Dynamic World classes:
//
// 0 = Water
// 1 = Trees
// 2 = Grass
// 3 = Flooded vegetation
// 4 = Crops
// 5 = Shrub and scrub
// 6 = Built
// 7 = Bare
// 8 = Snow and ice
//
// We use the same summer period as Landsat.
//

var dynamicWorld = ee.ImageCollection(
  'GOOGLE/DYNAMICWORLD/V1'
)
.filterBounds(studyArea)
.filterDate('2026-04-01', '2026-07-01');

print(
  'Dynamic World images:',
  dynamicWorld.size()
);


// ============================================================
// 8. CREATE SUMMER LAND-COVER COMPOSITE
// ============================================================

var dwLabel = dynamicWorld
  .select('label')
  .mode()
  .clip(studyArea);

Map.addLayer(
  dwLabel,
  {
    min: 0,
    max: 8
  },
  'Dynamic World Land Cover'
);


// ============================================================
// 9. CREATE BUILT-UP MASK
// ============================================================

// Dynamic World built-up class = 6

var builtMask = dwLabel.eq(6);

Map.addLayer(
  builtMask.selfMask(),
  {},
  'Built-up Areas'
);


// ============================================================
// 10. DEFINE URBAN AREA
// ============================================================
//
// To avoid calling the entire 60-km region "urban",
// we restrict urban built-up pixels to within 25 km
// of Kolkata.
//

var urbanZone = cityPoint
  .buffer(25000)
  .intersection(studyArea);

var urbanBuilt = builtMask
  .updateMask(
    builtMask
      .and(ee.Image.constant(1)
      .clip(urbanZone))
  );


// ============================================================
// 11. URBAN LST
// ============================================================

var urbanLST = summerLST.updateMask(
  builtMask.clip(urbanZone)
);

var urbanMean = urbanLST.reduceRegion({

  reducer: ee.Reducer.mean(),

  geometry: urbanZone,

  scale: 30,

  maxPixels: 1e9

});

print(
  'Urban mean LST:',
  urbanMean
);


// ============================================================
// 12. RURAL REFERENCE REGION
// ============================================================
//
// Rural reference area:
//
// 25–60 km from Kolkata
//
// We exclude:
// - built-up areas
// - water
//
// This gives us predominantly non-urban rural pixels.
//

var outerArea = studyArea.difference(
  cityPoint.buffer(25000)
);


// Non-built and non-water pixels
var ruralMask = dwLabel
  .neq(6)       // not built
  .and(dwLabel.neq(0));  // not water


var ruralArea = ruralMask
  .clip(outerArea);


// ============================================================
// 13. RURAL LST
// ============================================================

var ruralLST = summerLST
  .updateMask(ruralArea);

var ruralMean = ruralLST.reduceRegion({

  reducer: ee.Reducer.mean(),

  geometry: outerArea,

  scale: 30,

  maxPixels: 1e9

});

print(
  'Overall rural mean LST:',
  ruralMean
);


// Extract rural temperature
var ruralTemperature = ee.Number(
  ruralMean.get('LST')
);

print(
  'Rural reference temperature (°C):',
  ruralTemperature
);


// ============================================================
// 14. SUHI CALCULATION
// ============================================================
//
// SUHI = LST - rural reference temperature
//

var suhi = summerLST
  .subtract(ruralTemperature)
  .rename('SUHI');


// ============================================================
// 15. SUHI MAP
// ============================================================

var suhiVis = {
  min: -5,
  max: 10,
  palette: [
    'blue',
    'cyan',
    'white',
    'yellow',
    'orange',
    'red'
  ]
};

Map.addLayer(
  suhi,
  suhiVis,
  'Summer 2026 SUHI (°C)'
);


// ============================================================
// 16. URBAN SUHI STATISTICS
// ============================================================

var urbanSUHI = suhi
  .updateMask(builtMask)
  .clip(urbanZone);

var urbanSUHIStats = urbanSUHI.reduceRegion({

  reducer: ee.Reducer.mean()
    .combine({
      reducer2: ee.Reducer.median(),
      sharedInputs: true
    })
    .combine({
      reducer2: ee.Reducer.minMax(),
      sharedInputs: true
    }),

  geometry: urbanZone,

  scale: 30,

  maxPixels: 1e9

});

print(
  'Urban SUHI statistics:',
  urbanSUHIStats
);


// ============================================================
// 17. URBAN VS RURAL TEMPERATURE
// ============================================================

var comparison = ee.FeatureCollection([

  ee.Feature(null, {
    Area: 'Urban',
    Temperature: urbanMean.get('LST')
  }),

  ee.Feature(null, {
    Area: 'Rural',
    Temperature: ruralTemperature
  })

]);

print(
  'Urban vs Rural temperature:',
  comparison
);


// ============================================================
// 18. URBAN-RURAL TEMPERATURE CHART
// ============================================================

var chart = ui.Chart.feature.byFeature({

  features: comparison,

  xProperty: 'Area',

  yProperties: ['Temperature']

})
.setChartType('ColumnChart')
.setOptions({

  title: 'Summer 2026 Urban vs Rural Surface Temperature',

  vAxis: {
    title: 'LST (°C)'
  },

  hAxis: {
    title: 'Area'
  },

  legend: {
    position: 'none'
  }

});

print(chart);


// ============================================================
// 19. LST STATISTICS
// ============================================================

var lstStatistics = summerLST.reduceRegion({

  reducer: ee.Reducer.mean()
    .combine({
      reducer2: ee.Reducer.median(),
      sharedInputs: true
    })
    .combine({
      reducer2: ee.Reducer.minMax(),
      sharedInputs: true
    })
    .combine({
      reducer2: ee.Reducer.percentile([5, 25, 75, 95]),
      sharedInputs: true
    }),

  geometry: studyArea,

  scale: 30,

  maxPixels: 1e9

});

print(
  'Summer 2026 LST statistics:',
  lstStatistics
);
