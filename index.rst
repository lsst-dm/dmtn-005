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

python
^^^^^^
- cameraGeom

  - ``makePixelToTanPixel.py`` - this doesn't use wcs, but shouldn't it?
  - ``utils.py`` uses wcs for pixel shifting

- image

  - mostly swig output
  - ``utils.py`` has a few things to manage wcs.
  - ``basicUtils.py`` has a bunch of functions for comparing wcs

- table

  - mostly swig output

- math

  - ``warper.py`` uses them for exposure conversions

src
^^^
- detection

  - transformPoint and Footprint::transform - unclear, but should these be in here, or in image or math?

- formatters

  - ExposureFormatter/TanWcsFormatter/WcsFormatter - how to persist a wcs

- gpu

  - there's afw::gpu, but it's not clear to me whether it's being used properly.
  - math::detail::cudaLanczosWrapper wants to do wcs stuff?

- image

  - DistoredTanWcs - mostly not implemented
  - stuff for reading/writing fits files
  - makeWcs - where the magic happens?
  - TanWcs - where the magic happens
  - Wcs - where the magic really happens

- math

  - warpExposure extensive use of wcs for pixel position calculations/conversions

- table

  - Exposure reads/writes wcs and uses it to find whether a point is in the exposure.


ip_diffim
---------

python
^^^^^^
- Everything is in ip/diffim
- getTemplate uses pixelToSky() on the exposure's center/corners and skyToPixel() on the sky box.
- imagePsfMatch

  - imagePsfMatchTask.matchExposures only calls getWcs() while warping/making.
  - Lots of assumptions about wcs, but little explicit.
  - _validateWcs() compare if two exposures have same origin&extent. Used only in matchExposures() to check if we need to warp the template to the science.

- KernelCandidateQa.aggregate() takes wcsresids, but it's unclear what exactly those are (looks like a dict?)
- modelPsfMatchTask.run() uses getWcs to make an ExposureF
- utils.py

  - printSkyDiffs uses pixelToSky to compute deltas, though it looks rather odd.
  - makeRegions uses pixelToSky if a wcs is given, else just the source coordinates directly.
  - showSourcesSetSky uses skyToPixel to draw dots in ds9
  - plotWhisker uses pixelToSky to compute offsets between a set of astrometric matches.

src
^^^
- No references to Wcs at all in src!

examples
^^^^^^^^
- Several examples call warpExposure with exposure.getWcs() as the first arg, but that's nicely abstracted.
- imagePsfMatchTask generates a fake WCS as a FITS header.
- snapPsfMatchTask generates a fake WCS as a FITS header.

tests
^^^^^
- PsfMatchTestCases.makeWcs generates a fake WCS as a FITS header, which is what all the tests use to build their fake wcs.
- SnapPsfMatch.makeWcs generates a fake WCS as a FITS header, which is what all the tests use to build their fake wcs.


meas_astrom
-----------

python
^^^^^^
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

src
^^^
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

examples
^^^^^^^^
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

tests
^^^^^
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

 * Examples of the "standard" FITS projections, as implemented in astropy:
http://docs.astropy.org/en/stable/modeling/#module-astropy.modeling.projections
 * the papers describing those transforms:
http://adsabs.harvard.edu/abs/2002A%26A...395.1061G
http://adsabs.harvard.edu/abs/2002A%26A...395.1077C
