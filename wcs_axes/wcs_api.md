About
=====

A number of packages, such as pywcsgrid2, APLpy, and kapteyn, have developed
ways of making plots in world coordinate systems (WCS) in Matplotlib, and
since this is a common issue in Astronomy, we are interested in combining all
the common functionality for inclusion in the Astropy core. The present
document describes the proposed API.

The aim is not to provide the simplest possible user-friendly API to the
user, but to design an API consistent with the matplotlib API upon which
user-friendly tools can be developed.

Authors
=======

* Thomas Robitaille (@astrofrog)
* Jae-Joon Lee (@leejjoon)

Thanks for feedback from:

* Eli Bressert (@ebressert)
* Chris Beaumont (@ChrisBeaumont)
* Adam Ginsburg (@keflavich)
* Mike Droettboom (@mdboom)
* Phil Elson (@pelson)
* Moritz Guenther (@hamogu)
* Erik Tollerud (@eteq)

Requirements
============

The main requirements of the API are:

* Provide a ``WCSAxes`` class/projection which, given a WCS projection object,
  will automatically set up the infrastructure to show the relevant ticks,
  labels, and grid.

* The ``WCSAxes`` class should work with any arbitrary WCS, not just sky
  coordinates, but also physical quantities. In addition, it should in the
  long-term be able to work with the 'generalized' WCS being planned for
  Astropy.

* Make it easy for users to plot and overlay data in world coordinates, and
  in different coordinate systems.

* Support n-dimensional datasets

Usage
=====

Items marked *Convenience* would be lower priority, and not strictly required
for a functional API.

Initialization
--------------

The object-oriented interface (similar for ``plt.figure()`` and the true OO
API) uses a ``WCSAxes`` class which takes the same argument as Axes, but with
an additional argument for the WCS object.

    import matplotlib.pyplot as plt
    from astropy.wcs import WCS, WCSAxes

    fig = plt.figure()
    ax = WCSAxes(fig, [0.1, 0.1, 0.8, 0.8], wcs=WCS('image.fits'))
    fig.add_axes(ax)

If no WCS transformation is specified, the transformation will default to
identity, meaning that the WCSAxes object should then act like a normal Axes
object.

We can also provide standard Matplotlib axes initialization with the
``projection`` keyword:

    from astropy import wcs
    fig = plt.figure()
    ax = fig.add_subplot(1, 1, 1, projection=wcs.WCS('image.fits'))

or:

    from astropy import wcs
    fig = plt.figure()
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection=wcs.WCS('image.fits'))


This should also work with the interactive pyplot interface, and the
full-object oriented API.

Coordinate systems
------------------

As a convention, and to avoid any assumptions, if methods are called as:

    ax.method(...)

then the method applies to pixel coordinates. However, in general it will be
useful to be able to plot object in various world coordinate systems.
Matplotlib already supports plotting with arbitrary transformations, so a
method of ``WCSAxes`` will be available to give a transformation that can be
directly used by Matplotlib:

    tr_fk5 = ax.get_transform("fk5")  # returns a matplotlib.transforms.Transform instance

To plot in the FK5 system one would then do:

    ax.method(..., transform=tr_fk5)

To plot in the default world coordinates system, one should use:

    ax.get_transform("world")

By specifying a WCS object, one can also define a transformation from the
current WCS system to another file's WCS system, allowing e.g. overplotting of
contours in a different system:

    ax.get_transform(<WCS instance>)

If the world coordinate system of the plot is a celestial coordinate system,
the following built-in sky coordinate systems would be available from the
``get_transform`` method:

* ``'fk4'`` or ``'b1950'``: B1950 equatorial coordinates
* ``'fk5'`` or ``'j2000'``: J2000 equatorial coordinates
* ``'gal'`` or ``'galactic'``: Galactic coordinates
* ``'ecl'`` or ``'ecliptic'``: Ecliptic coordinates
* ``'sgal'`` or ``'supergalactic'``: Super-Galactic coordinates

Additional keywords may be required in these cases to specify e.g. observation
epochs and equinoxes.

Images
------

Plotting images as bitmaps or contours should be done via the usual
matplotlib methods (which can accept e.g. Numpy arrays or PIL Image objects
for RGB images)

    ax.imshow(fits.getdata('image.fits'))
    ax.imshow(Image.open('rgb_image.png'))
    ax.contour(fits.getdata('scuba.fits'))
    ax.contourf(fits.getdata('scuba.fits'))

Since WCS information technically doesn't require any information about
the image size, there would be no such thing as a 'mismatch' of dimensions
compared to the WCS.

In the case where we want to overplot an image as contours with a different
WCS projection, the coordinate system API described above would allow us to do
e.g.:

    ax.contour(scuba_image, transform=ax.get_transform(other_wcs))

Note that ``ax.imshow(..., transform=ax.get_transform(other_wcs))`` would not
work (due to matplotlib limitations).

*Convenience*: ``imshow``, ``contour``, and ``contourf`` should be able to
read in data and WCS from FITS and image files:

    ax.imshow('image.fits')
    ax.imshow('image_rgb.png')
    ax.contour('scuba.fits')

which should automatically overlay it in the right WCS. However, if ``imshow``
is given a FITS file with a WCS transformation different from the one being
used for the axes, then an exception should be raised.

Ticks and tick label properties
-------------------------------

While for many images, the coordinate axes are aligned with the pixel axes,
this is not always the case, especially in coordinate systems with high
curvature, where the coupling between x- and y-axis to actual coordinates
become less well-defined.

Therefore rather than referring to ``x`` and ``y`` ticks (as in APLpy), we
propose a new API, which allows very flexible customization of plots:

    # Access the first WCS coordinate
    ra = ax.coords[0]

    # Show ticks for this coordinate on the bottom and right axes
    ra.set_ticks_position('br')

    # Show tick labels only for the bottom axis
    ra.set_ticklabel_position('b')

    # Set the format of the tick labels
    ra.set_major_formatter('hh:mm:ss')

    # Set the spacing of the ticks
    from astropy import units as u
    ra.set_ticks_spacing(2. * u.arcmin)

and so on - we can also consider allowing aliases to certain coordinates, for
example:

    ra = ax.coords['RA--']

or even:

    ra = ax.coords['ra']

for common and recognized coordinate types.

Tick label format
-----------------

The format of the tick labels can be specified with the
``set_major_formatter`` (and optionally ``set_minor_formatter`` if desired),
that would either accept a standard Matplotlib ``Formatter`` object, or a
string describing the format:

    ra.set_major_formatter('x.xxx')  # decimal, non-angle coordinates,
                                      # 3 decimal places

    ra.set_major_formatter('d')  # decimal degrees, no decimal places

    ra.set_major_formatter('d.ddddd')  # decimal degrees, 5 decimal places

    ra.set_major_formatter('dd:mm')  # sexagesimal, 1' precision
    ra.set_major_formatter('dd:mm:ss')  # sexagesimal, 1" precision
    ra.set_major_formatter('dd:mm:ss.ss')  # sexagesimal, 0.01" precision

    ra.set_major_formatter('hh:mm')  # sexagesimal (hours)
    ra.set_major_formatter('hh:mm:ss')  # sexagesimal (hours)
    ra.set_major_formatter('hh:mm:ss.ss')  # sexagesimal (hours)

Tick/label spacing
------------------

The spacing of ticks/tick labels should default to something sensible, but
users often want to be able to specify the spacing of the ticks. The tick
positions could be set manually:

    ra.set_ticks([242.2, 242.3, 242.4])

but users can also set the spacing explicitly:

    ra.set_ticks(spacing=0.1)

or can set the approximate number of ticks:

    ra.set_ticks(number=4)

In the case of angles, it is often convenient to specify the spacing as a
fraction of a degree - users should then use the ``astropy.units`` framework:

    from astropy import units as u
    ra.set_ticks(5. * u.arcmin)

This is to avoid roundoff errors.

Note: allowing users to specify their own tick position or spacing, and the
label format raises the issue of what happens when the value cannot be well
represented by the format (this confused a lot of users in APLpy initially).
For example, if one specifies the format to be ``'dd:mm:ss'`` and a tick
spacing of 0.001 degrees then the ticks will go:

    13:42:00
    13:42:01
    13:42:03
    13:42:04
    13:42:06

and so on, which confuses users. In APLpy, we now raise an exception if the
tick spacing is not compatible with the labeling, but it might be best to
simply clip to the nearest value and emit an INFO message. If a tick spacing
is not specified, the minimum tick spacing should be set by the minimum one
representable by the format. For example, if the format is set to ``'dd:mm'``,
then the default tick spacing cannot end up being smaller than 1' (and if it
does it gets reset to 1').

Coordinate grid
---------------

Since the properties of a coordinate grid are linked to the properties of the
ticks and labels, grid lines 'belong' to the coordinate objects described
above. For example, one can show a grid with red lines for RA and blue lines
for declination with:

    ra = ax.coords['ra']
    dec = ax.coords['dec']
    ra.grid(color='blue')
    dec.grid(color='red')

and to modify the visual properties of the grid, users can capture a
handle to the grid lines:

    lines = dec.grid(color='red')

which is simply a ``LineCollection``, which can be modified:

    lines.set_alpha(0.5)
    lines.set_linewidth(2)

For convenience, users can also simply draw a grid for all the coordinates in
one command:

    ax.coords.grid()

There should also be an option to show the labels inside the axes instead of
on the side. This would be especially useful for all-sky projections.

Patches/shapes/lines
--------------------

To overlay arbitrary matplotlib patches in pixel coordinates:

    from matplotlib.patches import Rectangle
    r = Rectangle((43., 44.), 23., 11.)
    ax.add_patch(r)

To overlay arbitrary matplotlib patches in world coordinates:

    from matplotlib.patches import Rectangle
    r = Rectangle((272., -44.), 1., 0.8)
    ax.add_patch(r, transform=ax.get_transform('fk5'))

And similarly:

    ax.add_collection(c, transform=ax.get_transform('gal'))
    ax.add_line(l, transform=ax.get_transform('fk4'))

The same applies to normal plotting methods such as scatter/plot/etc:

    ax.scatter(l, b, transform=ax.get_transform('gal'))

Multiple coordinate systems
---------------------------

As shown above, the API could be set up to allow users to plot objects in
different coordinate systems, but only the default world coordinate system of
the file is used for the labels and ticks. To overlay a different coordinate
system to the one included in the file, one can do:

    glon, glat = ax.get_coords_overlay('galactic')

and then use the returned objects as the ``coords`` objects above, to set the
label, tick, and grid line properties.

To hide the labels, ticks, and grid lines for the default coordinate system,
one can do:

    ax.coords.set_visible(False)

Offset systems
--------------

In order to switch to showing offset rather than absolute coordinates, one can
use the coordinate objects described above:

    ra.enable_offset_mode(12.14)
    dec.enable_offset_mode(15.55)

or, more concisely,

    ax.coords.enable_offset_mode((12.14, 15.55))

where the tuple should have the same number of dimensions as the WCS. This
causes the coordinates to have the offset value subtracted, so in the above
case the point at ``(12.14, 15.55)`` would appear at ``(0, 0)``. To revert to
absolute coordinates:

    ax.coords.disable_offset_mode()

Multi-dimensional datasets
--------------------------

The WCS object can have more than two dimensions, but since we are ultimately
plotting two-dimensional data, we have to select which dimensions are used for
the x and y pixel coordinates in the axes, and we also have to select which
slices are selected for the dimensions that are not being shown.

We should try and implement this at the level of the WCS class - that is, it
should be possible to slice WCS objects and return a new WCS object with lower
dimensionality. One way to do this would be to have a ``WCS.slice()`` method
that would take a single 'slice' argument which should be specified as a list,
and where each element is either an integer (indicating the slice position) or
a string containing 'x' or 'y', indicating the dimension that should be used
for the x and y pixel coordinates. For example, if one does:

    wcs_new = wcs.slice(['x', 3, 'y'])

the first and third dimension will be used as the x and y pixel coordinates in
the Axes, and the second pixel coordinate will be set to 4 (could also be set
to a floating-point value). This could also be used on higher-dimension
datasets.

Examples
========

These are fake examples to show how would obtain a given plot with the
proposed API. It is a way to ensure that the specification above actually
works in practice.

Example 1
---------

![](https://github.com/astrofrog/astropy-api/blob/wcs-plotting-proposal-v2/wcs/lmc.png?raw=true)

    import matplotlib.pyplot as plt

    from astropy.wcs import WCS, WCSAxes
    from astropy.io import fits

    # Read in LMC image
    hdu = fits.open('lmc_mosaic.fits')

    # Initialize figure
    fig = plt.figure()
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection=WCS(hdu.header))

    # By default, the correct coordinate system will be shown on the x and y
    # axis, and the labels will be set based on the WCS coordinate system.

    # Add the colorscale
    ax.imshow(hdu.data, cmap=plt.cm.gist_heat, vmin=-1., vmax=10.)

    # Add the grid
    g = ax.coords.grid()
    g.set_color('white')
    g.set_alpha(0.25)

    # Save image
    fig.savefig('example1.png')

Example 2
---------

![](https://github.com/astrofrog/astropy-api/blob/wcs-plotting-proposal-v2/wcs/tutorial.png?raw=true)

    # This example assumes that a 3-color PNG of the Galactic center has been
    # previously made, and that the WCS information object is available in a
    # variable named `wcs_rgb`.

    import matplotlib.pyplot as plt

    from astropy.wcs import WCS, WCSAxes
    from astropy.io import fits
    from astropy.table import Table

    # Initialize figure
    fig = plt.figure()
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection=wcs_rgb)

    # By default, the correct coordinate system will be shown on the x and y
    # axis, and the labels will be set based on the WCS coordinate system.

    # Add the colorscale
    ax.imshow('galactic_center.png')

    # Overlay a contour from a FITS file with a different WCS
    ax.contour('msx.fits', levels=np.linspace(3., 10., 10), colors='white')

    # Overlay a catalog from coordinates in KF4
    cat = Table.read('source_catalog.tbl', format='ipac')
    ax.scatter(cat['GLON'], cat['GLAT'], edgecolor='red', facecolor='none', transform=ax.get_transform('gal'))

    # Save image
    fig.savefig('example2.png')

Example 3
---------

![](https://github.com/astrofrog/astropy-api/blob/wcs-plotting-proposal-v2/wcs/pgsbox4.png?raw=true)

    import matplotlib.pyplot as plt
    from astropy.wcs import WCS, WCSAxes

    # Initialize figure
    fig = plt.figure()
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection=WCS('image.fits'))

    # Set tick and label properties

    ra = ax.coords['ra']
    dec = ax.coords['dec]

    ra.hide_ticks()  # no ticks visible in plot
    ra.set_major_formatter('hh')
    ra.set_ticklabel_position('lb')  # left and bottom
    ra.set_axislabel_position('lb')
    ra.set_axislabel('Right ascension')
    ra.grid()

    dec.hide_ticks()  # no ticks visible in plot
    dec.set_major_formatter('dd:mm:ss.ss')
    dec.set_ticklabel_position('tr')  # top and right
    dec.set_axislabel_position('tr')
    dec.set_axislabel('Declination')
    dec.grid()

    # Save image
    fig.savefig('example3.png')

Example 4
---------

![](https://github.com/astrofrog/astropy-api/blob/wcs-plotting-proposal-v2/wcs/pgsbox5.png?raw=true)

    import matplotlib.pyplot as plt
    from astropy.wcs import WCS, WCSAxes

    # Initialize figure
    fig = plt.figure()
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection=WCS('image.fits'))

    # Set tick and label properties

    lon = ax.coords[0]
    lat = ax.coords[1]

    lon.hide_ticks()  # no ticks visible in plot
    lon.set_major_formatter('ddd')
    lon.set_ticklabel_position('tb')  # top and bottom
    lon.set_axislabel_position('tb')
    lon.set_axislabel_color('orange')
    lon.set_spacing(number=2)
    lon.grid(color='orange')

    lat.hide_ticks()  # no ticks visible in plot
    lat.set_major_formatter('ddd')
    lat.set_ticklabel_position('lr')  # top and right
    lat.set_axislabel_position('lr')
    lat.set_axislabel('latitude')
    lon.set_axislabel_color('blue')
    lon.set_spacing(30.)
    lon.grid(color='blue')

    # Set title
    ax.set_title("WCS conic equal area projection", color='aqua')

    # Save image
    fig.savefig('example4.png')

Example 5
---------

![](https://github.com/astrofrog/astropy-api/blob/wcs-plotting-proposal-v2/wcs/pgsbox7.png?raw=true)

    # We assume the original file is defined in Galactic coordinates

    import matplotlib.pyplot as plt
    from astropy.wcs import WCS, WCSAxes

    # Initialize figure
    fig = plt.figure()
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8], projection=WCS('image.fits'))

    # Set tick and label properties

    glon = ax.coords['glon']
    glat = ax.coords['glat']

    glon.set_major_formatter('ddd')
    glon.set_ticklabel_position('t')
    glon.set_ticklabel_color('green')
    glon.set_axislabel(None)
    glon.grid(color='green')

    glat.set_major_formatter('ddd')
    glat.set_ticklabel_position('r')
    glat.set_ticklabel_color('green')
    glat.set_axislabel(None)
    glat.grid(color='green')

    # Now show coordinate parameters for ecliptic grid

    elon, elat = ax.get_coords_overlay(system='ecliptic')

    elon.set_major_formatter('ddd')
    elon.set_ticklabel_color('orange')
    elon.grid(color='orange')
    elon.set_axislabel('longitude', color='orange')

    elat.set_major_formatter('ddd')
    elat.set_ticklabel_color('blue')
    elat.grid(color='blue')
    elat.set_axislabel('latitude', color='blue')

    # Set title
    ax.set_title("WCS plate caree projection")

    # Save image
    fig.savefig('example5.png')