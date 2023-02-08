
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

Provide a harness for full-focal-plane PSF estimation.
   We'll still probably do the second round of PSF estimation on each detector independently for the forseeable future, but we want to open up the ability to develop and test a full-focal-plane algorithm.

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
A lot of these are obvious and probably don't need to be listed here, but we should try to track anything that has caused problems that led to restructuring in the past.

- Aperture corrections must be applied before we can compute "extendedness" for star-galaxy classification.


.. _pipeline-realizations:

Pipeline Realizations
=====================

Each subsection below is a different proposal for the DRP pipeline, and should include some discussion of how and why it differs from other realizations.

Diagrams should generally leave out dataset types that don't participate in the relationships between tasks (e.g. calibrations, reference catalogs).

The Original: December 2020
---------------------------

This is the pipeline we ran in Gen2, imagined as a complete Gen3 pipeline, given what I know about the tasks that are still being converted.
I've included the tasks all the way through ``compareWarpAssembleCoadd``, because realizations of the future pipeline will often want to feed the masks it generates back into single-frame processing steps.

Those of you who haven't been following the FGCM Gen3 conversion may be surprised to see it taking ``sourceTable_visit`` and ``visitSummary`` as inputs and write visit-level catalogs of ``PhotoCalibs`` as outputs, but that solves a number of problems in Gen3 and would probably be a good move in Gen2 if it was worth our time to make the change there, too.
I (JFB) was surprised to see that the former is the *standardized* parquet table, because that means we definitely can't apply the FGCM calibrations before or when making the standardized parquet table, but given that there's no unstandardized visit-level parquet dataset (i.e. ``source_visit``), that makes sense right now.
But it might make more sense to consolidate ``source`` -> ``source_visit`` and have FGCM consume that instead, even before we consider the other changes proposed in this technote;  we'd then standardize ``source_visit`` -> ``sourceTable_visit`` directly and apply calibrations at that point.

I'm mostly assuming Jointcal will use those same visit-level inputs and similarly have a visit-level ``ExposureCatalog`` as its output, and I've talked to John P. a bit about that, but it's possible I've guessed wrong about some of those details.

The names of tasks in the diagram below are the ``_DefaultName`` attributes for those tasks, and should also be used as their labels in the Gen3 pipeline definition.
One exception is ``SkyCorrectionTask``; its ``_DefaultName``, ``skyCorr``, is the same as the name of its main output dataset type, so I've used ``correctSky`` in the diagram for the task instead.

Finally, *wow*, we need some dataset type naming conventions.
Let's think about that a bit as we design the future pipeline, and maybe by the time we're ready to start implementing it we'll have some ideas on how to better name things.

.. figure:: /_static/the-original.svg
    :name: pl-the-original

JFB's Ambitious No-Fakes Pipeline, v1: December 2020
----------------------------------------------------

This is JFB's first attempt at writing down a future pipelines that meets all of our goals, except for the ones involving adding fake sources.

There's a ton to describe and explain here, but I'll start with the *simplifying* assumptions I've made, which actually keep this from being bigger than it is.

- I've assumed that we can perform some kind of compensated filter photometry quite early (in ``characterizeImage``), to feed FGCM with fluxes that aren't corrupted by background subtraction.
  And I've assumed that calibrating *those* fluxes is enough to calibrate later photometry that doesn't use compensated filters but can take advantage of better backgrounds.

- I've assumed that we can use a catalog of high-detection-threshold stars (more or less what we detect in today's version of ``characterizeImage``) for *all* image characterization and calibration steps, so we can put off full-depth detection/deblending/measurement until all of those steps are complete.

- I've only tried to write down a global-sequence-point version of FGCM, and I've rolled that up into one task instead of several.
  Think of this as just drawing a box around the FGCM sub-graph, not declaring that FGCM shall be one task.

And now some "ambitious" decisions I've made across the board; these generally simplify the pipeline itself, while probably complicating our transition to it.
I've gone in that direction here because I'd prefer to think about the end state independently of the transition, but that may not wise; I'm betting we will want some "less ambitious" or transitional pipeline diagrams.
Anyhow on that note,

- I've assumed we'll just convert from afw.table to DataFrame/parquet *immediately* at the end of the ``PipelineTasks`` (just ``characterizeImage`` and ``finishVisitImage``) that perform measurement, and never write out ``afw.table.SourceCatalog`` objects (or if we do, it'll just be for ``Footprints``, and nothing will consume them).

- I've assumed ``jointcal`` (or ``GBDES``; I'm guessing that's what we'll be using by the time we get there, but I've still used "jointcal" as the name of the step below) and ``FGCM`` can be made to use the same matched catalog, and we can do that matching up-front, once.

- I've assumed we'll want to run our initial, single-visit photometric and astrometric calibration on full-visit catalogs, not visit-detector catalogs.  That should be a bit more robust, but it may or may not be something AP will want to do, depending on how big the robustness improvement is relative to the compute latency hit.
  It also simplifies the pipeline description, just because it reduces the number of consolidate-to-visit steps.

Now some nomenclature/conventions:

- I've started using ``snake_case`` for dataset type names (while keeping ``camelCase`` for task labels), just to make it extra clear which is which.

- I've started using ``pvi`` instad of ``calexp``.

- I've tended to add prefixes to dataset type names when they aren't the final version, and remove those prefixes (or use ``pvi_`` as a prefix) when they are the final versions.

- I've left off dependent dimensions, just for brevity (i.e. if "visit" or "detector" is present, we don't need to say "instrument").

You'll also note that "tracts" don't appear in this pipeline until we get all of the way to coaddition, and instead you'll see "skypix[1-4]" as the spatial dimension for a lot of previous steps.
These refer to HEALPix (preferably) or HTM (if necessary) grids at particular TBD depths, with roughly the following scales:

skypix1
   Used for matched ``ic_src`` catalogs; intended to have cells much larger than a visit, so most visits are wholly contained in one cell.

skypix2
   Scale at which ``jointcal`` runs; cells are intended to be no larger than those of skypix1 and no smaller than a visit.

skypix3
   Scale at which ``prepRevisit`` runs (see below); cells are no larger than those of skypix2, and no smaller than a detector.

skypix4
   Used for matched ``src`` catalogs.  Same size arguments as skypix1, as far as I can tell, and probably is just the same as skypix1.

I've pretty much punted on some possibly important stuff here:

- Fakes, obviously.

- I think we'll need to re-do aperture corrections after we have our final PSF model, but it seems like that will need to be tied up with subtracting bright star wings and tying FGCM's calibration back to our final non-compensated-filter photometry.

- I've halfheartedly tried to include background matching, as a ``comparison_bkg`` dataset output by a coaddition step, but it probably isn't a realistic depiction.

And finally, a brief description of each step.
I won't repeat the dimensions or the inputs and outputs here, so this section is definitely best read while constantly referring to the diagram below.

isr
   The same ISR we know and love, or whatever it evolves into independent of the plans of this technote.

characterizeImage
   Essentially the same ``CharacterizeImageTask`` we have today, though there are some parts I'd like to streamline.

consolidateIcSrc
   Just gathers the per-detector catalogs into a visit-level one.

consolidateVisitSummary
   Just gathers various per-detector Exposure components into a visit-level ``ExposureCatalog``.
   Unlike what we have today, this doesn't deal with ``PhotoCalib`` or ``Wcs`` yet, because those don't exist yet.

bootstrapAstrometry
   Matches to a reference catalog and fits for initial WCSs, over a full visit.  Also outputs a thin table of sky coordinates, with the same rows as its input catalog.

bootstrapPhotometry
   Matches to a reference catalog and fits for initial photometric calibration, over a full visit.  Also outputs a thin table of calibrated fluxes, with the same rows as its input catalog.

matchIcSrc
   Multi-way match of all visit-level catalogs and one or more external reference catalogs (at least Gaia) over a large area of sky (skypix1).
   Outputs normalized match indices (``ic_src_match_join``) and a summary table (``ic_src_match``).

fgcm
   At least a few rounds of ``fgcmFitCycle``.
   Probably starts with FGCM making its own star table from the outputs of ``matchIcSrc``; hopefully it would start from the same matches, so what it uses would be a straightforward subset of those.
   The LUT is not made here - that's more of a CPP pipeline thing, I think.
   In addition to its own model and per-visit photometric calibration datasets, this also outputs a thin table of calibrated fluxes and inferred colors for each input source match (``ic_src_match_fluxes``).

jointcal
   Multi-epoch astrometry fitting, at first implemented by today's jointcal package, but eventually backed by GBDES.
   Outputs both WCSs for all input visits and a thin table of stellar motion parameters for the bright-ish matched stars it was given.
   A separate task will compute stellar motion parameters for more deeper detections later.
   I'm assuming this will eventually do some internal DCR correction on its input positions, using the colors it gets (via ``ic_src_match_fluxes``) from FGCM.

prepRevisit
   A catalog-space operation that works out what we'll want to do on each detector-level image when we return to it shortly, based on what it sees in the matched catalogs.
   This includes selecting PSF stars (and reserving some) and figuring out what data we'll want to extract from the images in order to fit the wings of bright stars.
   It's entirely possible we'll actually make this multiple ``PipelineTasks``.

extractRevisit
   Return to the pixel data, one visit+detector at a time, and extract summary information we can use for visit-level catalog-space fitting.
   For example, this might extract postage stamps for PSF estimation, a binned image for background estimation, and *something* for wings-of-stars fitting.

fitRevisitPsf
   Fit the final PSF model for an entire visit, using already-extracted postage stamps.

fitRevisitBackground
   Fit the wings of bright stars and a large-scale smooth background (after unsubtracting the original small-scale background from ``characterizeImage``), using the summary statistics from ``extractRevisit`` and whatever CPP products (e.g. something like today's ``sky`` frames) it likes.

planCoaddition
   A catalog-space operation that computes exactly which visits will go into each coadd *and their weights*, taking into account large-scale masks, PSF-size or -quality rejection, etc.
   Doing this up front - and not per-patch - seems to be the only way to weigh visits consistently over the entire tract, and avoid discontinuities in the per-patch coadd images.
   It also gives us a chance to figure out in advance what target PSFs each visit-level image will need to be matched to when we make warps.
   It's quite possible that we'll actually want to make this multiple versions of the same PipelineTask, with each corresponding to a different final coadd dataset type.

makeComparisonWarp
   Essentially today's ``MakeWarpTask``, configured for making the PSF-matched warps that ``compareWarpAssembleCoadd`` will want to use.

compareWarpAssembleCoadd
   Initially this is today's ``CompareWarpAssembleCoaddTask``, but with an explicit output mask dataset.
   Eventually I imagine this step doing background matching and producing a background output as well, but I don't know enough about background matching to know if that actually fits in this structure.

finishVisitImage
   Finally, we run detection at full depth (5-sigma threshold), redeblend, and remeasure, after applying all of the previous image characterization models we've produced, including the transformed backgrounds and masks from ``compareWarpAssembleCoadd``.
   We also match to the previous ``ic_src_match`` catalog to propagate flags from those steps.
   Because this is the first measurement we've run with our latest PSFs, we also remeasure our aperture corrections, which might actually be a quite different kind of aperture correction, if it needs to tie us back to the FGCM measurements.

consolidateSrc
   Just gathers the per-detector catalogs into a visit-level one.
   Same task as ``consolidateIcSrc``, just reconfigured.

matchSrc
   Multi-way match of all visit-level catalogs and one or more external reference catalogs (at least Gaia) over a large area of sky (skypix4).
   Outputs normalized match indices (``src_match_join``) and a summary table (``src_match``).
   Same task as ``matchIcSrc``, just reconfigured.

fitSrcMatchAstrometry
   Fit stellar motion parameters for all matched ``src_match`` rows, holding the
   WCSs fixed.


.. figure:: /_static/jfb-ambitious-nofakes-01.svg
    :name: pl-jfb-ambitious-nofakes-01


The Great Calibration Refactor Proposal: February 2023
------------------------------------------------------

Goals and non-goals
"""""""""""""""""""

We're replacing essentially everything between ISR and coaddition.
ISR and coaddition will also see major changes (spurred by Calibpalooza and cell-based and chi-squared coadds, respectively) on similar timescales (we hope), but we're considering those out-of-scope and mostly orthogonal.
While we can and should implement this piecemeal when we can, we want a complete vision of what it will look like in the end, and in some cases it may be easier to replace many tasks at once.

We lean towards merging PipelineTasks with the same dimensions that run back-to-back rather than keeping them distinct.
This is a bit of a shift - many smaller PipelineTasks leads to more flexibility via just pipeline definition changes, which has been very useful in prototyping, but we believe we are exiting the prototyping phase and should instead prioritize the I/O optimization and pipeline-simplicity advantages of having fewer bigger PipelineTasks.
We very much intend to continue to delegate all real algorithmic work to subtasks; it's just that each PipelineTask will tend towards having more of those.

This is probably our last best chance to get our naming conventions for dataset types and task labels under control, so we're including that in our proposal.

Conventions
"""""""""""

- Task labels are camelCase and start with a lowercase verb: "associateIsolatedStars" instead of "isolatedStarAssociation".
- Dataset type names are snake_case nouns preceded (if necessary) by adjectives: e.g. "initial_visit_summary"
- Use "source" instead of "src" or "sources".
- Avoid "catalog" or "cat" in dataset type names; use "source", "stars", "object", or "matches" instead when appropriate.
- Use some variant of "pvi" for any direct (non-difference) ``{visit, detector}`` image dataset with an image, mask, and variance plane (i.e. ``lsst.afw.image.Exposure`` or ``MaskedImage``).
- Tables that are initially per-detector that will get concatenated into per-visit tables should get a "_detector" suffix so the final thing does not need a suffix (and when we revamp later steps of the pipeline, the same for "_patch" so there's no "_tract").
- Task labels and dataset type names are for "slots", not specific tasks or connections - usually those are 1-1, but when they are not, the label and dataset type names should remain fixed when a different task is swapped in (our "solveAstrometry" task label slot could be satisfied by either jointcal or GBDES).
  Whether the same is true of the outputs is an open question we'd like to discuss.
- "initial" catalogs are not (ever) SDM-standardized, while final catalogs are always SDM-standardized before they are written out.
  This may present a challenge for analysis tools that want to be able to run on either, but I'm hoping we can minimize analysis of the initial things, as we don't care how good they are in the end.
- Do include a "final" prefix on dataset types that represent the best version of things that are nevertheless temporaries that will not be retained.
  Do not add any prefix to dataset types that will be retained for public access.
- Convert source and object catalogs to Parquet before ever persisting them; only use SourceCatalog/FITS to hold Footprints.
  We will retain the ``slot_*`` column names and not the underlying names they point to - downstream code should only be referring to the slot columns anyway, and in many cases we won't run multiple algorithms that could satisfy the slot.
  In addition, note that the (final) ``source`` catalogs will be SDM-standardized before being written, so the the ``slot_*`` vs. underlying name question is moot there.

Task and dataset type notes
"""""""""""""""""""""""""""

bootstrapImage
   Initial background subtraction and detection of bright stars (galaxies are considered a nuisance here).
   Initial versions of everything - astrometry, photometry, probably PSFs.
   Probably aperture corrections of some kind, but targeted specifically at making compensated apertures work for FGCM.
   These are all attached to its image output, ``initial_pvi``, which is a lot like today's ``calexp``.
   This will be a fluence image with nJy pixel units, but how much of that is done by this task vs. ISR is TBD.
   Its output catalog, ``initial_source_detector``, will be converted to Parquet before it is written, but not SDM-standardized.
   Footprints will be written to a separate ``SourceCatalog`` dataset, ``initial_footprints``.
   Whether this task does multiple detection rounds (to iterate on CR detection, the source detection filter, or the detection threshold) is TBD; we'd like to minimize that.
   We would like to avoid running a deblender here (if we do, it will have to happen after we have the PSF model), but this depends on how this performs on crowded fields.
   We are not thrilled with the name of this task and would love ideas for improvements (which should be coordinated with the names for ``bootstrapVisit``, ``finalizeImage``, ``compressImage``, and ``rebuildImage``).

bootstrapVisit
   Consolidate the per-detector outputs of ``bootstrapImage`` and recover from failures on some detectors by using those that succeeded (especially for WCSs).

associateIsolatedStars
   Pretty much just a renamed ``IsolatedStarAssociationTask``.

solveAstrometry
   Pretty much just a generic label for ``jointcal`` and ``GBDES``.

fgcm
   Nothing new here, except the sharding and names of the output datasets.

modelVisitBackground
   A replacement for SkyCorrectionTask, probably using PCA.
   SkyCorrectionTask will serve as a placeholder until we have something better.
   Also takes care of subtracting the wings of bright stars.

finalizeCharacterization
   Nothing new here except new names for the output datasets.

   We are using "characterizations" here to mean "PSFs and aperture corrections", and are not thrilled about that, but we need something that means "PSFs and aperture corrections" that could also absorb aperture corrections being done rather differently than they are today.

finalizeAstrometry
   This is a re-run of the astrometry solver with slightly different connections; I think it'll be necessary to make our (achromatic) WCSs consistent with the chromatic PSF models that we'll someday produce in ``finalizeCharacterization``.
   But this needs more thought.

updateVisitSummary
   Nothing new here (though the current task is only a few weeks old, and apparently still has some bugs).

finalizeImage
   This new task takes all of our hard-won final characterizations and calibrations of the image and produces a final ``{visit, detector}`` ``pvi`` image and full-depth, all-measurements ``final_source_detector`` catalog.
   The latter will be SDM-standardized before it is ever written to disk.

consolidateSourceTable
   This task just concatenates the ``final_source_detector`` catalogs into a single per-visit ``source`` catalog.

compressImage
   This task reads the ``pvi`` dataset, lossy-compresses the image and variance planes, lossless-compresses everything else (at least the mask; I don't know if compressing more than that is possible).

rebuildImage
   This task reconstructs the uncompressed PVI from the compressed one.
   It needs to be preceded by rerunning ISR, and then it just subtracts the (retained) ``visit_background`` and pulls the lossless-compressed (or uncompressed) mask plane and components from the compressed PVI.

.. figure:: /_static/great-calibration-refactor.svg
    :name: pl-great-calibration-refactor


Major Questions
===============

What are the open questions that drive the differences between different viable pipeline structures?

- Can we feed FGCM with compensated-filter photometry in order to run it before background estimation is complete?

- Do we need some kind of coaddition or warp-comparison to finalize our per-visit background models?  If so, will those also impact PSF modeling and/or aperture corrections?

- Is a second round of astrometric fitting necessary to ensure PSF chromaticity is consistent with our (achromatic) WCSs?  Is it sufficient?

- Are there small changes we could make to better share code with AP?

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
