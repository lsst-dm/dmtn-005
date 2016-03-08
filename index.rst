..
  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

.. warning::
  This technical note is currently a draft!


:tocdepth: 1

======================================
Current usage of WCS in the lsst stack
======================================

The products afw, ip_diffim, and meas_astrom each make extensive use of our Wcs
implementation. In other packages, the use is mostly in tests or examples.


afw
---

afw/python
^^^^^^^^^^
cameraGeom/makePixelToTanPixel.py
  Doesn't use wcs, but shouldn't it?
cameraGeom/fitsUtils.py
  makeWcs from metadata and make exposure with it.
cameraGeom/utils.py
  prepareWcsData() converts from amp to ccd coordinates, calls [1] flipImage,
  [2] shiftReferencePixel.
  makeFocalPlaneWcs creates a basic px->mm Wcs via CR/CD/etc. and calling makeWcs.
display/interface.py
  mtv can take a wcs if you don't pass an exposure, for display.
display/simpleFits.cc
  Output a WCS FITS image file descriptor for e.g. DS9.
display/utils.py
  drawCoaddInputs calls skyToPixel(pixelToSky) for every ccdCorner, for display.
image
  mostly swig output
image/basicUtils.py
  Compare Wcs in a grid. Loops over xyList calling pixelToSky, then skyToPixel.
image/utils.py
  getDistortedWcs() creates a DistortedTanWcs from a tanWcs+pixelsToTanPixels.
image/table
  mostly swig output
math/warper.py
  Takes Wcs to convert between exposures. computeWarpedBBox() calls
  skyToPixel(pixelToSky) for the edges of the bounding box. Calls mathLib for the
  real work of warpExposure and warpImage.

afw/src
^^^^^^^
detection/Footprint::transformPoint and Footprint::transform
  Transform point x,y from one image to another via skyToPixel and pixelToSky,
  and use transformPoint to transform corners and loop over points.
  Should these be in here, or in image or math?
ExposureFormatter/TanWcsFormatter/WcsFormatter
  How to persist a wcs.
gpu
  There's afw::gpu, but it is only referenced in afw and the implemetation is several years old: should it be dropped?
  math::detail::cudaLanczosWrapper imports Wcs.h, but doesn't seem to do anything with it.
image
  DistoredTanWcs - mostly not implemented!
  Exposure.cc, ExposureInfo.cc, Image.cc, Mask.cc have utilities for reading/writing Wcs to FITS.
  makeWcs - Produce a WCS object from a FITS header.
  TanWcs - where the magic happens
  Wcs - where the magic really happens
math
  warpExposure extensive use of wcs for pixel position calculations/conversions
table
  Exposure.cc reads/writes wcs and uses it to find whether a point is in the exposure.
  Source.cc updateCoord uses pixelToSky on either the centroid or a given center.

afw/examples
^^^^^^^^^^^^
timeWarpExposure.py
  Test several interpolation lengths, (one) scale factor, (one) arcsec offset,
  rotation angle and kernel name, making a Wcs and timing how long it takes to warp an exposure with it.
warpExposure.py
  All wcs handling is managed internally Exposures.
timeWarpGpu.cc
  Timing of GPU acceleration, calls warpImage with the Wcs's.
wcsTest.cc
  Demo of outputting various computed values from a FITS file with a WCS.

afw/tests
^^^^^^^^^
sipterms.cc
  Tests linearWcs vs. sipWcs.
testWarpGpu.cc
  Timings of gpu accelerations, plus some actual comparison tests between GPU and CPU.
testWcs.cc
  Direct number comparisons of Wcs calculations.
\*.py
  Plenty of python tests.


ip_diffim
---------

ip_diffim/python
^^^^^^^^^^^^^^^^
getTemplate.py
  Calls pixelToSky on an exposure's center/corners and skyToPixel() on the sky corners.
imagePsfMatch.py
  imagePsfMatchTask.matchExposures only calls getWcs() while warping/making.
  Lots of assumptions about wcs, but little explicit.
  _validateWcs() compare if two exposures have same origin&extent. Used only in
  matchExposures to check if we need to warp the template to the science.
  Calls pxielToSky on bounding boxes.
KernelCandidateQa.py
  aggregate() takes wcsresids, but it's unclear what exactly those are (looks like a dict?)
modelPsfMatchTask.py
  uses exposure.getWcs to make an ExposureF.
utils.py
  printSkyDiffs calls pixelToSky to compute deltas, though the call signature looks odd.
  makeRegions uses for loop over sources to call pixelToSky if a wcs is given, else just the source coordinates directly.
  showSourcesSetSky uses for loop over sources to call skyToPixel to draw dots in ds9.
  plotWhisker uses pixelToSky to compute offsets between a set of astrometric matches.

ip_diffim/src
^^^^^^^^^^^^^
No references to Wcs at all in src!

ip_diffim/examples
^^^^^^^^^^^^^^^^^^
Several examples call warpExposure with exposure.getWcs() as the first arg, but that's nicely abstracted.

imagePsfMatchTask
  Generate a fake WCS as a FITS header.
snapPsfMatchTask
  generates a fake WCS as a FITS header.

ip_diffim/tests
^^^^^^^^^^^^^^^
PsfMatchTestCases.makeWcs
  generates a fake WCS as a FITS header, which is what all the tests use to build their fake wcs.
SnapPsfMatch.makeWcs
  generates a fake WCS as a FITS header, which is what all the tests use to build their fake wcs.


meas_astrom
-----------

meas_astrom/python
^^^^^^^^^^^^^^^^^^
anetAstrometry.py
  uses hasDistortion(), shiftReferencePixel(), skyToPixel(), pixelToSky() and
  calls makeCreateWcsWithSip()
anetBasicAstrometry.py
  Uses updateCoord(wcs) to update a source catalog. Also calls pixelScale(),
  pixelToSky(), isFlipped(), linearizePixelToSky(), skyToPixel(),
  getFitsMetadata(), shiftReferencePixel().
approximateWcs.py
  Either calls getSkyOrigin(), getPixelOrigin(), and getCDMatrix to then use
  makeWcs() to generate a tanWcs, or just uses a tan wcs directly. Calls
  makeCreateWcsWithSip() on said tanWcs.
astrometry.py
  Calls pixelToSky(). Defaults to using fitTanSipWcsTask to do the fit.
fitTanSipWcs.py
  Calls skyToPixel() and pixelToSky(). Instantiates afwImage.Wcs from coordinates.
matchOptimisticB.py
  All the work happens in the src lib, though there is one call to pixelScale().
sip/cleanBadPoints.py
  Calls skyToPixel, though appears to be broken? Only applies to X array.

meas_astrom/src
^^^^^^^^^^^^^^^
makeMatchStatistics.cc
  makeMatchStatisticsInPixels/makeMatchStatisticsInRadians statistics of on-
  sky/detector given a wcs and a list of matches. Use skyToPixel and pixelToSky,
  respectively.
matchOptimisticB.cc
  Several functions that call skyToPixel/pixelToSky, e.g. between tangent and
  distorted Wcs's. matchOptimisticB() uses wcs.hasDistortion() to check and
  build a tanWcs using wcs.getCDMatrix() on the distorted one.
CreateWcsWithSip.cc
  Computes SIP distortion between catalogue and image, given the matches and a
  linear Wcs from image pixels to catalog RA/Dec. Instantiates afw::image::Wcs
  and afw::image::TanWcs. Depends on getPixelOrigin, getCDMatrix,
  skyToIntermediateWorldCoord, undistortPixel, getSkyOrigin.
  Claims to use Wcs to to convert xy <->ra/dec to find common objects between
MatchSrcToCatalogue.ccf
  source and image lists. Appears to do this via image->updateCoord(wcs), as the
  wcs is not used elsewhere.

meas_astrom/examples
^^^^^^^^^^^^^^^^^^^^
getSourceSet.py
  ``makeCcdMosaic()`` creates a wcs from FITS metadata. ``showStandards()`` gets
  a wcs from an image and uses skyToPixel to check whether standards are in the
  image. ``setRaDec()`` calls pixelToSky to   set ra/dec for each source in a
  list.
imsimPlots.py
  Creates a TanWcs from the Wcs of a calexp, and plots them with wcsPlots.
rerun-wcs.py and rerun_wcs.py
  Creates a wcs from determineWcs and writes it to a fits file via
  wcs.gtFitsMetadata(). NOTE: the '_' version is nearly a superset of the '-'
  version, but not entirely...
ticket2710.py
  Why isn't this a test? Creates a few Wcs and calls their skyToPixels().
wcsPlots
  Used by some of the above to plot matches, using skyToPixel()

meas_astrom/tests
^^^^^^^^^^^^^^^^^
CreateWcsWithSip.py
  Calls pixelToSky() and skyToPixel(). Some commented out FITS code, and an updateCoord call.
openFiles.py
  testDetermineWcs and testUseKnownWcs don't actually test anything, but call a
  function 3+ times each! The OpenFilesTest docstring claims this is intended
  behavior...
testAstrometryTask.py
  Instantiates afwImage.TanWcs from FITS metadata, to build an image and afwImage.Instantiates DistortedTanWcs in the test.
testFitTanSipWcsHighOrder.py
  Instantiates afwImage.TanWcs from FITS metadata. Instantiates DistortedTanWcs in tests, and has code to plot the Wcs.
testFitTanSipWcsTask.py
  Makes a tanWcs from raw numbers and its pixelToSky(), skyToPixel(). Also has plotting code.
testLoadAstrometryNetObjects.py
  Instantiates afwImage.Wcs from FITS metadata and uses its pixelToSky()
testMakeMatchStatistics.py
  Instantiates afwImage.TanWcs from FITS metadata.
testMatchOptimisticB.py
  Calls afwImage.makeWcs from FITS metadata. Calls skyToPixel() and pixelToSky(). Instantiates afwImage.DistortedTanWcs() in a test.
testSetMatchDistance.py
  Calls afwImage.makeWcs from raw numbers and its pixelToSky().
testSipTransformations.py
  Calls afwImage.makeWcs from values in files and their pixelToSky(), skyToPixel().

pipe_tasks
----------

- many examples in the docs.
- calibrateTask.py uses it in an example
- coaddBase uses it in SelectDataIdContainer.makeDataRefList
- mockObservation builds simple WCSs
- testRegister does a bunch of wcs things
- wcsSelectImages does a bunch of wcs stuff, but it might all be tests.


Other uses
----------

daf_butlerUtils
  used to make an ExposureFromImage
meas_algorithms
  used in several tests
meas_extensions_psfex
  wcs get built in psfex for ds9 display
meas_modelfit
  makes wcs for XY transforms (one wcs to another) in UnitSystem.cc
obs_lsstSim
  genInputRegistry.py uses it to convert an image to a polygon
obs_sdss
  A few things use it for image conversions/parsing (all python)
skymap
  All of the BaseSkyMap-derived python classes use it.
coadd_chisquared
  Coadd.py class takes an lsst.afw.math.Wcs;
  chisquaredLib_wrap.cc refers to afw__image__\*Wcs stuff.
coadd_utils
  utilsLib_wrap.cc refers to afw__image__\*Wcs stuff;
  Coadd.py class takes an lsst.afw.math.Wcs
ip_isr
  assembleCcdTask.py gets/sets wcs from exposures.

Other notes
-----------

So I don't lose other things I've found that may be relevant later:

Examples of the "standard" FITS projections, as implemented in astropy:

| http://docs.astropy.org/en/stable/modeling/#module-astropy.modeling.projections
|

The papers describing those transforms:

| http://adsabs.harvard.edu/abs/2002A%26A...395.1061G
| http://adsabs.harvard.edu/abs/2002A%26A...395.1077C
