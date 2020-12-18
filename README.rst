.. image:: https://img.shields.io/badge/dmtn--172-lsst.io-brightgreen.svg
   :target: https://dmtn-172.lsst.io
.. image:: https://github.com/lsst-dm/dmtn-172/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-172/actions/
..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

##########################################################
Multi-Stage Image Characterization and Calibration for DRP
##########################################################

DMTN-172
========

This technical note is an informal collection of outlines, diagrams, and commentary centered around an effort to restructure and extend DRP's pre-coaddition processing, now that the Gen3 middleware has made those kinds of changes significantly easier.  It is intended to be a living document used for collaborative design work by the DRP team, not a polished description of a complete design.

**Links:**

- Publication URL: https://dmtn-172.lsst.io
- Alternative editions: https://dmtn-172.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-172
- Build system: https://github.com/lsst-dm/dmtn-172/actions/

**To add a new pipeline realization,**

- Open a ticket and start a new branch of the technote.
- Make a new ``.dot`` file in the root directory of the technote (probably by copying an existing one that similar to the one you want).
- Add a new entry to the ``FIGURES`` variable in the ``Makefile``, using the same base filename as the name of the ``.dot`` file.
- Start a new subsection under "Pipeline Realizations", and add a figure entry that references the diagram (again, see existing realizations for an example).
- Update the "Goals", "Invariants", and "Major Questions" if your pipeline realization addresses a new or old problem, or makes different assumptions about what will work in the future.
- Send the review to Jim.

New pipeline realizations that are just slight modifications of existing ones are quite welcome - if the technote gets long, that's fine.
The in-text descriptions of these pipelines can just highlight the diffs from the one they started from - they don't need to describe each step again.
But they should have their own complete diagram.


Build this technical note
=========================

You can clone this repository and build the technote locally with `Sphinx`_:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-172
   cd dmtn-172
   pip install -r requirements.txt
   make html

.. note::

   In a Conda_ environment, ``pip install -r requirements.txt`` doesn't work as expected.
   Instead, ``pip`` install the packages listed in ``requirements.txt`` individually.

The built technote is located at ``_build/html/index.html``.

Editing this technical note
===========================

You can edit the ``index.rst`` file, which is a reStructuredText document.
The `DM reStructuredText Style Guide`_ is a good resource for how we write reStructuredText.

Remember that images and other types of assets should be stored in the ``_static/`` directory of this repository.
See ``_static/README.rst`` for more information.

The published technote at https://dmtn-172.lsst.io will be automatically rebuilt whenever you push your changes to the ``master`` branch on `GitHub <https://github.com/lsst-dm/dmtn-172>`_.

Updating metadata
=================

This technote's metadata is maintained in ``metadata.yaml``.
In this metadata you can edit the technote's title, authors, publication date, etc..
``metadata.yaml`` is self-documenting with inline comments.

Using the bibliographies
========================

The bibliography files in ``lsstbib/`` are copies from `lsst-texmf`_.
You can update them to the current `lsst-texmf`_ versions with::

   make refresh-bib

Add new bibliography items to the ``local.bib`` file in the root directory (and later add them to `lsst-texmf`_).

.. _Sphinx: http://sphinx-doc.org
.. _DM reStructuredText Style Guide: https://developer.lsst.io/restructuredtext/style.html
.. _this repo: ./index.rst
.. _Conda: http://conda.pydata.org/docs/
.. _lsst-texmf: https://lsst-texmf.lsst.io
