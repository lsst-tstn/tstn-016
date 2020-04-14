..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.



:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

Introduction
============

This document describes how to determine the sensitivity matrix for performing alignment and collimation using the
Curvature Wavefront Sensing (CWFS) alignment technique described technote `tstn-015 <tstn-015.lsst.io>`_. A sensitivity
matrix (in this case) is used to relate the measured wavefront error (WFE) to the offsets of the hexapod holding
the secondary mirror such that the WFE is corrected. In the case of this telescope, the only correction factors
being performed are the centering of the hexapod (in x and y) and the defocus (piston in z). This technote walks through
the derivation of such a matrix, from the optical model estimates, then on-sky testing and verification.

One of the largest challenges of this exercise is getting the coordinate relationships correct, specifically in regards
to the X/Y offset directions of the hexapod versus what they are on the detector. To add further complications is how
python uses a [Y,X] array format combined with how displays manage those images (e.g. matplotlib, firely, ds9).
If one repeats this exercise they must pay strict attention to these relationships.

The format of the formula relating hexapod offsets to measured Zernike Annular (Noll) coefficients is as follows in
the :ref:`equation below <eqn_matrix>`, where
the zernikes are derotated to the boresight frame. This is where the elevation and nasmyth frames are locked in
rotation, meaning 90 degrees elevation is zero degrees of the instrument rotator, and similarly 60 degrees of
elevation is +30 degrees of elevation.

.. _eqn_matrix:

.. math::

    \begin{bmatrix}
    dx \\
    dy \\
    dz
    \end{bmatrix}
    =
    \begin{bmatrix}
    Z7 & Z8 & Z4
    \end{bmatrix}
    \begin{bmatrix}
    C_{X}       & C_{YX}/C_{Y} & C_{ZX}/D_{Z} \\
    C_{XY}/C_{X} & C_{Y}       & C_{ZY}/D_{Z} \\
    C_{XZ}/C_{X} & C_{YZ}/C_{Y} & D_{Z}
    \end{bmatrix}


The sensitivity matrix itself is in units of millimeters (of motion of the hexapod) per nm (of measured WFE of a given
Zernike polynomial.

.. Important::
    The sensitivity matrix is for hexapod motion only. In order to build (useful) look-up tables the relationship
    between M1 mirror pressure versus elevation must first be derived as it strongly affects the results, primarily
    the defocus term.


.. _optical_model:

Optical Model Output
====================

The first approximation of the sensitivity matrix can be accomplished using the optical model of the
telescope (in Zemax,
`Document-27702 <https://docushare.lsst.org/docushare/dsweb/Get/Document-27702>`_, credentials required). This was
performed and it successfully measured the defocus relationship quite accurately, as well as the coma terms. This was
accomplished by taking the above optical model and adding a coordinate break at the secondary mirror in order to
decenter. The effect of defocusing the secondary was also added by modifying the distance between M1 and M2 and M2 and
M3 by equal amounts.

.. figure:: /_static/optical_model_coma_WFE_estimation_screenshot.jpg
    :name: zemax_screenshot

    The sensitivity matrix can be determined from the optical model, but sign errors will most certainly result when
    trying to convert to image space. Elements require verification in an incremental fashion.

The terms of the sensitivity matrix in the  :ref:`equation above <eqn_matrix>`
are found from offsetting the hexapod (in decenter and piston), then
looking at the annular Zernike coefficients. Because the system is linear for first order terms offsets of 1 mm were
performed. The diagonal coma terms from optical model is 168 nm/mm (+1mm decenter in X is Z8=-178nm, +1 mm
decenter in Y is Z7=-178nm) whereas the cross-terms (:math:`R_{23}` and :math:`R_{32}`) are zero. Note that the X-axis
here is as per Zemax conventions.
The defocus term is significantly more sensitive
due to the powered optics, and is estimated at 4.85 nm/um. Seeing as the magnification between the motion of the
secondary mirror and the back focal distance is ~44, this is not unexpected.

.. Important::
    In the model above, it is assumed that there are no tilts in the system and that all optics are perfect. Having
    cross-terms are possible due to tilts in the system, but these must be measured experimentally.


Data Acquisition
================

This section discusses how to derive the experimental measurement of the sensitivity matrix. It is assumed that
one starts from the matrix derived in section :ref:`optical_model`, which is purely diagonal. The general procedure
of the test consists
of aligning the system to the best of one's ability at a given elevation, then stepping the hexapod in small
steps and measuring the WFE. Performing the test successfully requires a little more thought.

1. Due to the numerous rotations in the system, it is best to choose a target that is transiting the meridian
(will not greatly change in elevation) over the course of ~1.5-2 hours. Then, align the system to the best of one's
ability, which will take several iterations. Choose a target near elevation 60.
2. It is strongly recommended to keep the
elevation and rotator axes aligned to the extent possible (meaning the difference between rotator angle and elevation
angle should be kept nearly constant. This means adjustments will be required after each pair of images.
3. Perform the test later in the night, when the dome seeing has been reduced. Exposures should be ~30s long. This
will require a star of magnitude ~6-8
4. Use a bandpass filter to reduce chromatic effects.

The hexapod can then be offset using relative offsets from the attcs class as shown in the following code block.
Relative offsets are generally preferred as it does not require the AOS loop to be turned off.

.. code-block:: python

    offset = {'x':0, 'y':  0.0, 'z': 0.0}
    await attcs.ataos.cmd_offset.set_start(**offset)

If you get lost or want to reset the offsets, this can be done as follows.

.. code-block:: python

   await attcs.ataos.cmd_resetOffset.start()

Data Reduction
==============

One of the later datasets to measure the cross-terms was performed on 2020-02-18 on HD 27583.
The data reduction and derivation of the matrix terms are best described from the accompanying jupyter notebook, which
is found :ref:`here <>`.

The output of the notebook shows the linear fits applied to the measured Zernike coefficients, the slope of which can
be input into the sensitivity matrix. The first fit shows coma as a function of offset, one would expect this to
follow the optical model quite closely.

.. figure:: /_static/Y_coma_as_x_fxn_of_Y_displacement.jpg
    :name: Y-Coma plot

    The slope of this fit is used in the FIXME term of the sensitivity matrix.

The next plot shows the derivation of the cross-term between a decenter and defocus. Due to the large magnification
in the system, any mis-alignment will be heavily amplified.

.. figure:: /_static/Defocus_as_x_fxn_of_Y-displacement.jpg
    :name: Defocus plot

    The slope of this fit is used in the FIXME cross-term of the sensitivity matrix. Why this term is so large is
    a bit of a mystery.

A cross-term of 106 nm of defocus for 1 mm of lateral hexapod motion is the equivalent of 22um of hexapod motion.
This is equivalent to having the hexapod mounted with a tilt of 1.26 degrees. This is ~3x larger than what one
would expect from mechanical alignment therefore we suspect (but cannot confirm) that M3 is tilted and is not
at exactly 45 degrees.

One can also look at the relationship between Y-hexapod motion and X-Coma. One would assume it should be largely zero.

.. figure:: /_static/X-coma_as_x_fxn_of_Y-displacement.jpg
    :name: X-Coma plot

    This can be used in term FIXME of the sensitivity matrix, but is omitted as it's significantly smaller than the
    others and probably within the measurement errors due to turbulence.


Applying the Fits to the Sensitivity Matrix
===========================================

The sensitivity matrix is currently only in latiss_cwfs_align.py script. In order to use another matrix, either that
file must be modified, or the notebook calling it can modify it using the following code snippet:

.. code-block:: python

   from lsst.ts.externalscripts.auxtel.latiss_cwfs_align import LatissCWFSAlign
   script = LatissCWFSAlign(index=1, remotes=False) # Change to True if taking live images
   script.sensitivity_matrix = [
        [1.0 / 161.0, 0.0, 0.0],
        [0.0, -1.0 / 161.0, (107.0/161.0)/4200],
        [0.0, 0.0, -1.0 / 4200.0]
        ]

.. note::
    At time of writing, the summit was shutdown and we are running the scripts/notebooks on the LSP at NCSA. There
    are known differences in the outputs between when we ran things on the summit and what we get at NCSA. Once the
    summit comes back online we should be able to resolve these differences.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
