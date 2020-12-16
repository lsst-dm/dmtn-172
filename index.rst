..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This technical note is an informal collection of outlines, diagrams, and commentary centered around an effort to restructure and extend DRP's pre-coaddition processing, now that the Gen3 middleware has made those kinds of changes significantly easier.  It is intended to be a living document used for collaborative design work by the DRP team, not a polished description of a complete design.

Goals
=====

What do we want to accomplish by reorganizing our pre-coaddition processing?

Let's keep this list focused on things the current processing *can't* do for structural reasons (or has to work around unpleasantly for structural reasons), and order it roughly by importance (i.e. things that are definitely problems now come before things that might be problems later).

Use a consistent set of stars for PSF estimation on all overlapping visits (and reserve a consistent set for cross-validation).
  We will want this catalog to include colors, a star sample with higher purity than we can expect to achieve from classification on individual visits alone, and consistent positions (e.g. average sky centroids projected back through per-visit WCSs).
  That means this needs to happen after some amount of cross-visit matching and astrometric fitting of at least bright and preferably isolated stars has  happened.

Make a final catalog of single-epoch detection measurements, utilizing the best image characterization and calibration models we will produce.
  Any image characterization or calibration that happens after this round of detection/deblending/measurement must be something we can apply sufficiently well in catalog-space that we're not tempted to go back to the pixels again.

Measure astrometric motion parameters for matched sources, down to the single-epoch detection limit.
  These will have to be matched to the Object catalog later.
  This need not be done when we fit jointly for the best WCS for each single-epoch image - it could be done later, if that initial fit only uses a brighter, more pure sample of stars - but there may be limitations on the processing flow imposed by the astrometry fitter(s) we use.

Make use of masks from warp-comparison (including satellites) in producing final single-epoch catalogs.
  This can probably be done in catalog-space in terms of setting flag columns, but it may be that these masks are important for doing some of the background estimation we'd want to do prior to final single-epoch detection/deblending/measurement.

Avoid workarounds from background-induced photometry biases in FGCM.
   This shouldn't be FGCM's job, and it shouldn't be something that only happens inside FGCM.
   It is unclear, however, whether the best solution is to run FGCM only after the final measurement step (after the final background-estimation step), or to try to devise some compensated-filter photometry that is insensitive to the background (for stars only) that we could run earlier, allowing FGCM to be run earlier.

Distinguish between steps that are common to DRP and AP, and steps that are DRP-only.
   This involves at least avoiding unnecessary-to-AP steps in the earliest stages of processing, in order to keep the performance of those steps viable for AP.  Ideally ISR and the first few steps immediately after it could be exactly the same, but we should recognize that this may not be possible, and some steps may need to be differently configured or ordered, and instead focus on how best to share common logic.

Limit the number of possible points at which we support inserting fakes, and limit the extra fakes-only processing steps needed to make use of those fakes.
   This should only affect the non-fakes pipeline if there really is no other constraint on how to order the affected steps, though.
   This is a challenge because we need to run the fakes only after some visit models (e.g. PSF, photometric calibration, astrometric calibration) are finalized, but at least some use cases for fakes want them run *before* others (e.g. background subtraction) are finalized.

Limit the number of cross-visit matched catalogs whenever possible.
   This should reduce confusion and make it easier to interpret analyses and metrics derived from them.
   For example, if joint astrometric and photometric fitting and repeatability metrics *can* use [subsets of] the same one or two catalogs, they should.


Invariants to Preserve
======================

What are the subtle (and of hard-won) properties of the current pipeline structure that we need to preserve?
A lot of these are obvious any probably don't need to be listed here, but we should try to track anything that has caused problems that led to restructuring in the past.

- Aperture corrections must be applied before we can compute "extendedness" for star-galaxy classification.


.. _pipeline-realizations:

Pipeline Realizations
=====================

Each subsection below is a different proposal for the DRP pipeline, and should include some discussion of how and why it differs from other realizations.
Qualitatively new tasks needed by this pipeline should be added to the :ref:`Tasks section<tasks>`, but slight modifications to already-described tasks can just be included directly in the pipeline realization description.


.. _tasks:

Tasks
=====

Each subsection below describes a concrete ``PipelineTask``.
Some of these already exist in the codebase, while others will need to be added.

IsrTask
-------

Instrument signature removal; already present in ``ip_isr``.

**Dimensions:** instrument, detector, exposure

**Inputs:**

- ``raw`` (Exposure): instrument, detector, exposure
- (various calibrations)

**Outputs:**

- ``postISRCCD`` (Exposure): instrument, detector, exposure

CharacterizeImageTask
---------------------

Shallow single-epoch processing, focusing on PSF estimation and aperture corrections; already present in ``pipe_tasks``.

**Dimensions:** instrument, detector, visit

**Inputs:**

- ``postISRCCD`` (Exposure) : instrument, detector, exposure

**Outputs:**

- ``icExp`` (Exposure): instrument, detector, visit
   - adds ``psf`` component
   - adds ``apCorrMap`` component
   - background subtracted
   - CRs repaired

- ``icExpBackground`` (Background): instrument, detector, visit

- ``icSrc`` (SourceCatalog): instrument, detector, visit
   - High SNR sources only.

**Notes**

This task *implicitly* includes snap-combination today, because it's where the dimensions transition from "exposure" to "visit".
We actually just don't run it on any data with more than one snap.
We could also consider adding another ``PipelineTask`` that runs before it, responsible for just combining snaps.
I don't think we can really consider moving snap-combination to ISR, at least not if we want to keep using ISR in CPP (which has nothing to do with visits).

We should try to drop the iterative PSF determination, detection, and measurement.
That really shouldn't be necessary, and the version of me that once thought it was seems young and foolish today.
But we might instead want something that would be more robust in the presence of crowded fields.

We should definitely drop the illusion that there can be any reference-catalog loading or matching here; that has never worked, and never will.

Measurement includes slow algorithms like CModel, both for aperture corrections and single-epoch extendedness; need to trim these for AP.

CalibrateTask
-------------

Full-depth single-epoch processing, focusing on (single-detector) astrometric and photometric calibration; already present in ``pipe_tasks``.



Major Questions
===============

What are the open questions that drive the differences between different viable pipeline structures?



.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
