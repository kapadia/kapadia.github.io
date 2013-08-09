---
layout: post
title:  "Volume Renderings"
date:   2013-08-02
categories: visualization
---

# Power of the browser.
# Exposing data to the browser, now what?
# HDR images (using libraries such as WebFITS)
# Tables (using libraries such as D3)
# Data Cubes?

As the browser becomes a first class platform, many are developing full blown web applications that rival desktop applications.  Libraries such as `fitsjs` expose rich data to the browser, allowing for these data to exist in the browser ecosystem.  With astronomical data exposed in the browser, we can use the variety of software the exist in the web ecosystem.

For instance, importing a FITS table to the browser is achieved using `fitsjs`, and creating interactive plots is achieved using D3.  Astronomers now have their data, in its native format, exposed so that these tools may be used.

But what else can be done in the browser?  Keeping on the topic of FITS files, we have a variety of data to choose from.  FITS images are easily rendered using WebFITS, but how about cubes?


FITS files store various data types, including images and tables.  Tapping into these data within the browser means popular JavaScript libraries may be used to visualize the information.  

, one of which are three-dimensional images.  Traditionally these data have been visualized by flipping through each slice individually.  Rather than viewing one slice at a time, volume renderings provide a way to visualize the entire cube at once, exposing the three-dimensional form of the data.

To visualize these data the images must be stacked and extruded to form a cube.  Volume raycasting is used to create the rendering, a technique borroewed by those working in the medical field.

The library is called `volumetric.js`.  



`fitsjs` has been refactored to handle more real-world use cases. Originally the library was developed to read images for only Zooniverse use cases.  This meant reading small SDSS and HST cutouts of around 500 pixels.  Most users of FITS (pretty much just astronomers and a few folks at the Vatican Library) handle much larger files.  With this refactoring, `fitsjs` handles files of around 1 gigabyte or more in size.

With the update, `fitsjs` has improved read speeds utilizing multi-threading (via Web Workers), and has a more javascript-esque API, heavily relying on callback functions for time intensive operations.

Find [`fitsjs`](https://github.com/astrojs/fitsjs) and [documentation](http://astrojs.github.io/fitsjs/) on GitHub.

The library is easy to use, almost like PyFITS, but with callback functions.  Here's a snippet:
    
    // Define a callback function for when the file is read
    function readCallback(f) {
      
      // Get the first header
      var header1 = f.getHeader();  // or f.getHeader(0)
      
      // Get the second header if the file has multiple header-dataunits
      var header2 = f.getHeader(1);
      
      // Print a keyword from header
      console.log(header.get('BITPIX'));
      
      // Get the first dataunit
      var dataunit1 = f.getDataUnit();
      
      // Or how about the second dataunit
      var dataunit2 = f.getDataUnit(1);
      
      // Assuming this is an image, let's get the pixels (it's a asynchronous process)
      dataunit1.getFrame(0, function(arr) {
        console.log(arr);
      });
    }
    
    // Initialize
    var f = new astro.FITS('/path/to/fits/file.fits', readCallback);
