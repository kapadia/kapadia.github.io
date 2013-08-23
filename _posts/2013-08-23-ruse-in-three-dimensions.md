---
layout: post
title:  "Ruse In Three Dimensions"
date:   2013-08-23
categories: visualization
---

A few weeks ago I started working on a WebGL based plotting library.  The software is still in its infancy, but it's beginning to show potential.  Here's a [quick demo of Ruse plotting three dimensional data](http://astrojs.s3.amazonaws.com/ruse/examples/three-dimension-plot.html).

The primary benefit of a project like Ruse is scalability.  Existing tools are cumbersome when handling larger datasets.  Packages such at [matplotlib](http://matplotlib.org/) render using the CPU, whereas Ruse utilizes the GPU.  With most modern browsers supporting WebGL, there's no need to install OpenGL bindings to use, for instance, a Python based OpenGL plotting package.

Ruse is still missing key components, like the absent axes in the above demo, but these items will be added in the next few weeks.

Find [`ruse.js`](https://github.com/kapadia/ruse.js) on GitHub.


