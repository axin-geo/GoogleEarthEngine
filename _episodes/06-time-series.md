---
title: "Time Series"
teaching: 5
exercises: 10
questions:
- How do I create a time series for a given location?
- How can I plot that time series within Google Earth Engine?
- "How do I make that plot interactive?"
objectives:
- "Load MODIS imagery and calculate snow index."
- "Plot timeseries of average snow index by counties in WA"
- "Dynamically select lat/longs for creating time series plots."
- "Create a time series of NDSI for a selected point."
keypoints:
- "Time series data can be extracted and plotted from Image Collections for points and regions."
- "GEE has increasing functionality for making interactive plots."
- "The User Interface can be modified through the addition of widget."

---

## Overview

This code allows users to dynamically generate time series plots for from points that are dynamically chosen on a map on the fly. The time series show the 8 day composites of Normalized Difference Snow Index at 250 m resolution. These indices are derived from MODIS.

Link to a static version of the full script used in this module:
[https://code.earthengine.google.com/a47b635ed6a11a99199674364afb9944](https://code.earthengine.google.com/a47b635ed6a11a99199674364afb9944)

## Define User Specifications

This script is structured to make it easy for the user to select different images, dates and regions. For this exercise, we are going to leave the parameters as they are to set the extent as a study area in Skagit County, Washington.

{% highlight javascript %}
// load county boundaries and select Skagit County in Washington
var counties = ee.FeatureCollection('ft:1ZMnPbFshUI3qbk9XE0H7t1N5CjsEGyl8lZfWfVn4');
// state number for WA: 53, county number for Skagit: 57
var roi = counties.filter(ee.Filter.eq('STATEFP',53)).filter((ee.Filter.eq('COUNTYFP',57)));

{% endhighlight %}
<br>

## Load MODIS Surface Reflectance and Calculate NDSI

NDSI is a spectral index calculated from green and shortwave-infrared bands. Here, we are deriving our own NDSI from MODIS 8 day composite surface reflectance product using the `normalizedDifference` function. NDSI ranges from -1 to 1, with values greater than 0 generallyconsidered to have snow present.

{% highlight javascript %}

// load modis 8 day surface reflectance and filter to 2017
var Modis = ee.ImageCollection('MODIS/006/MOD09A1')
  .filterDate('2017-01-01','2017-12-31')
  
// write function to calculate snow index using band 4 (green) and band 6 (SWIR)
var calcNDSI = function(img) {
  var ndsi = img.normalizedDifference(['sur_refl_b04','sur_refl_b06']).rename('NDSI')
  return img.addBands(ndsi)
}

// map function over modis collection
var NDSI = Modis.map(calcNDSI).select('NDSI')

// visualize maximum snow index from throughout the year and add region of interest above snow index layer
var maxSnow = NDSI.max()
Map.addLayer(maxSnow, {min: -1, max: 1})
Map.addLayer(roi);
Map.centerObject(roi,7);
{% endhighlight %}


## Make plot of average NDSI by region

Here we make a plot of 8-day spatially averaged NDSI in Skagit County.   

 {% highlight javascript %}

// Visualize ----------------------------------------------------------------------------------
Map.centerObject(setExtent, 8);

// A nice EVI palette
var palette = [
  'FFFFFF', 'CE7E45', 'DF923D', 'F1B555', 'FCD163', '99B718',
  '74A901', '66A000', '529400', '3E8601', '207401', '056201',
  '004C00', '023B01', '012E01', '011D01', '011301'];

// greenest images
Map.addLayer(annualGreenest.clip(setExtent).select('GI_max_14'),
    {min:.75, max: 15, palette: palette}, 'GI - annual');

// cdl and naip
Map.addLayer(cdl, {}, 'cdl', false);
Map.addLayer(naip, {}, 'naip', false);
{% endhighlight %}

<br>
<img src="../fig/06_annualGreennest.png" border = "10">
<br><br>

## Create a User Interface

You can alter the client-side user interface (UI) through the ui package by adding widgets to the Code Editor interface. You can read about the ui package in the [UI Overview section](https://developers.google.com/earth-engine/ui) of the Developers Guide
The general idea is that you make a widget, which could be simple (a button) or complex (a chart). Then you define the behavior of the widget and then add it to the display. Here we create a panel, define the contents of the panel using labels, and create a callback function so the user can click a point and it will record the lat/long as an object called `points.` You can read more about how to define the panel and layouts in the [Panels section](https://developers.google.com/earth-engine/ui_panels) of the Developers Guide.

{% highlight javascript %}

// Create User Interface portion --------------------------------------------------
// Create a panel to hold our widgets.
var panel = ui.Panel();
panel.style().set('width', '300px');

// Create an intro panel with labels.
var intro = ui.Panel([
  ui.Label({
    value: 'Two Chart Inspector',
    style: {fontSize: '20px', fontWeight: 'bold'}
  }),
  ui.Label('Click a point on the map to inspect.')
]);
panel.add(intro);

// panels to hold lon/lat values
var lon = ui.Label();
var lat = ui.Label();
panel.add(ui.Panel([lon, lat], ui.Panel.Layout.flow('horizontal')));

// Register a callback on the default map to be invoked when the map is clicked
Map.onClick(function(coords) {
  // Update the lon/lat panel with values from the click event.
  lon.setValue('lon: ' + coords.lon.toFixed(2)),
  lat.setValue('lat: ' + coords.lat.toFixed(2));
  var point = ee.Geometry.Point(coords.lon, coords.lat);
  {% endhighlight %}


## Add the time series plots to the panels

Now that we have set up our user interface and built the call-back, we can define a time series chart. The chart uses the lat/long selected by the user and builds a time series for NDVI or EVI at that point. It takes the average NDVI or EVI at that point, extracts it, and then adds it to the time series. This series is then plotting as a chart.


  {% highlight javascript %}

  // Create an MODIS EVI chart.
  var eviChart = ui.Chart.image.series(collectionModEvi, point, ee.Reducer.mean(), 250);
  eviChart.setOptions({
    title: 'MODIS EVI',
    vAxis: {title: 'EVI', maxValue: 9000},
    hAxis: {title: 'date', format: 'MM-yy', gridlines: {count: 7}},
  });
  panel.widgets().set(2, eviChart);

  // Create an MODIS NDVI chart.
  var ndviChart = ui.Chart.image.series(collectionModNDVI, point, ee.Reducer.mean(), 250);
  ndviChart.setOptions({
    title: 'MODIS NDVI',
    vAxis: {title: 'NDVI', maxValue: 9000},
    hAxis: {title: 'date', format: 'MM-yy', gridlines: {count: 7}},
  });
  panel.widgets().set(3, ndviChart);
});

Map.style().set('cursor', 'crosshair');

// Add the panel to the ui.root.
ui.root.insert(0, panel);
{% endhighlight %}

You should see something like this appear in the bottom left:

<br>
<img src="../fig/06_twoChart.png" border = "10" width="75%" height="75%">
<br><br>

## Exploring the NDVI and EVI plots for different crop types.

Toggle the Greenness, CDL and NAIP imagery layers on and off. Use the inspector to click on pixels with different levels of Greenness or different crop types and explore the differences in the NDVI time series between different pixels.



## Extracting Time Series Data for larger regions or more points

If you are computing the indices on the fly, or you have many points or areas of interest, you may have the unpleasant experience of your code timing out. One way to avoid that is to just export the time series as a .csv to Google Drive or Cloud Storage. An example of how to do this can be found in [Episode 04: Reducers](https://geohackweek.github.io/GoogleEarthEngine/04-reducers/) of this tutorial.
