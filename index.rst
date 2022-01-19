:tocdepth: 1

.. sectnum::

Abstract
========

This document proposes a relatively simple mechanism for using PanDA and Rucio to execute the Data Release Production across multiple sites while minimizing the transfer of data.
Possibilities for future development are also described.


Background
==========

The Data Release Production needs to run at multiple sites.
50% of the processing will be performed at the French Data Facility (FrDF) at CC-IN2P3.
25% of the processing will be performed at the UK Data Facility (UKDF), which may actually make use of resources at multiple physical locations in the UK.
25% of the processing, as well as the central control, will be hosted at the US Data Facility.

The USDF and FrDF will host 100% of the raw images and the published data products.
It is expected that the UKDF will only host 25% of the raw images.
It may host a larger fraction of the published data products, but we should not rely on 100% of those products being present at the UKDF.
The files or objects composing these raw and published data products, along with temporary data products, are referred to here as "datasets", corresponding with LSST Data Butler terminology.
Each Butler dataset is typically composed of a single file that constitutes a usable scientific entity.
In some cases, however, complex datasets with multiple components may be persisted as more than one file.
Note that this somewhat conflicts with standard Rucio terminology, in which a "dataset" is a collection of files.

All sites will run the same Science Pipelines code, including middleware, using specifically-versioned binary artifacts retrieved from CVMFS.
All sites will have a local Data Butler repository with Registry and Datastore using a PostgreSQL database.
But the underlying batch execution system and storage systems may vary.
In particular, the Datastore is typically expected to be an object store but might be a shared filesystem.

We are expecting to use Rucio to transfer data (raw, some intermediate, and published data products) from location to location.

We are writing a Replica Monitor service that monitors Rucio to determine when a replica of a dataset has been created and automatically ingests that dataset into a local Butler.
Note that for this Replica Monitor to work, there must be sufficient metadata information either within the replicated file, replicated alongside, or perhaps registered as Rucio file metadata to allow Butler ingest.
This metadata includes the Butler dataset type, dataId dimension values, and collection name.
It also includes any provenance metadata pertaining to the dataset.

We are expecting to use the Batch Production Service (BPS) software in conjunction with PanDA to define and control the execution of processing workflows at each Data Facility.

We are expecting to use a custom Campaign Management system to control the overall submission of workflows to Data Facilities and the sequencing of those workflows as part of the Data Release Production.


Existing Single-Site Processing
===============================

At a single site, an operator (or developer) provides a pipeline YAML file, a source Butler, and some configuration parameters to BPS.
BPS accesses the source Butler to create a Quantum Graph (QG) describing the workflow to be performed, including all of the datasets to be used, both those already existing and those to be created.

An Execution Butler (EB) is generated from the Quantum Graph.
This is materialized as a SQLite database that is provided to each job in the workflow; it holds all the relevant Butler Registry and Datastore information for the workflow, obviating the need to contact the source Butler for each job.
Each job retrieves datasets from or persists datasets to the underlying Datastore using pre-computed URIs in the EB.
Depending on the type of URI (shared filesystem or object store), these datasets may be copied and cached locally on the worker node executing the job.

After execution of all the jobs in the workflow, the EB is merged into the source Butler, making the results available and preserving provenance.


Extension to Multi-Site Processing
==================================

The Butler Datastore tracks local datasets; it does not maintain knowledge of replicas in different locations.
This is good, as it means that Rucio is the source of truth for which datasets are where.

Rather than trying to create a workflow that spans sites, we will insist that each workflow be executed at exactly one site.
This simplifies the execution model, and it deterministically guarantees that potentially large temporary datasets are not implicitly transferred between sites.
It will be the responsibility of Campaign Management to ensure that appropriate workflows are dispatched to appropriate sites and that all workflows constituting the Data Release Production are eventually executed at some site.
Typically a region of the sky would be assigned to a site, and all workflows associated with that region would be directed to that site.

It is acknowledged that PanDA is typically used in a data-driven mode in which multi-site workflows are submitted with PanDA deciding where a given job should execute based on the location of prerequisite datasets.
We are not using this mode because the Data Release Production is likely to use extensive temporary files derived from the Processed Visit Images (PVIs) that are shared between multiple coadd jobs within the same patch.
Since these would be temporary, relatively large, and relatively numerous, they should not be replicated across sites nor regenerated for each coadd patch job.

If the use of such temporaries can be internalized within a given job (which might, for example, work on all PVIs overlapping a patch at once rather than processing PVIs and patches separately), then it may be possible to loosen the single-site workflow constraint.

Another alternative might be to register all temporaries in Rucio without replication, but this could significantly impact Rucio scalability.

.. figure:: /_static/Multi-Site-BPS.png
    :name: fig-multi-site-bps

    Multi-Site Processing with BPS.

For the single-site workflow, we can continue to have the pipeline YAML file, source Butler, and configuration parameters be provided to BPS.
The source Butler is located at the USDF.
As the EB is created, it needs to contain URIs local to the site where the workflow will be executed.
There are two possible ways to do this: either the source Butler can contain "Rucio URIs" that are site-independent and the EB creation can use Rucio APIs to translate these into site-specific URIs, or the source Butler can contain USDF-local Datastore URIs and the EB creation can know how to translate these into the site-specific URIs.
The latter is undesirable, as it requires detailed knowledge of the specifics of the USDF storage.

The resulting EB and job descriptions (including the final merge job) are handed to PanDA as is currently done for single-site execution, but in this case the site may be remote from the USDF.
As jobs execute, they produce result datasets in the local Datastore of the site.
When the EB merge job runs, it merges into the local Butler at the site as usual, but it also has an additional step that registers datasets of particular dataset types with Rucio.
(It may be possible to only do the Rucio registration and leave the Butler ingest to the automated Rucio monitor, but this is likely to be less efficient than having the monitor only process true replicas rather than initial ingests.)
The dataset type list is composed of all published data products as well as any temporary data products that need to be globally distributed or globally summarized.
The merge step does not communicate with the source Butler at the USDF.

As datasets are registered in Rucio, Rucio takes charge of replicating them to appropriate destinations (all Data Facilities for globally distributed datasets and the USDF for globally summarized datasets).
The Replica Monitor then ingests the results into local Butlers at each site, making them available for use by following workflows and jobs.
In particular, it ingests them into the source Butler at the USDF for future QG generation.
This means that workflows that require the outputs of a preceding workflow must wait until Rucio has replicated all of the replicable products to the USDF before beginning QG generation.
This latency is not expected to be high (but should be measured), and given the amount of processing to be done can usually be filled with other workflows that need to be executed.
However there is a use case (executing a workflow based on failed jobs in a previous workflow) where such latency may be undesirable.


Future Improvements
===================

Distributed Rucio Query
-----------------------

If the translation step in the EB generation is a bottleneck due to having it being done serially in a single job, distributing this to the workers by having the EB be in terms of "Rucio URIs" might seem to be a possibility.
However those workers still need to contact the central Rucio servers at the USDF over transatlantic links, so it seems difficult for this to be more efficient.
In addition, Rucio provides a batched ``/replicas/list`` interface that seems likely to make EB generation sufficiently efficient.

Site-local QG generation
------------------------

Since the site-local Butler Registry and Datastore have all needed information about locally-present datasets, they could be used to generate QGs for workflows submitted to the site.
Since the URIs in its Datastore are already site-local, no translation step would be needed.

Essentially this would be using Campaign Management to do single-site workflow execution at each site independently, although Rucio, Rucio registration in the merge job, and the Replica Monitor are still necessary to replicate outputs.

One complication with this model is determining how workflow submission to PanDA (or the underlying site batch system) would be done.
If a global PanDA submission is desired to allow centralized tracking of all workflows, then the QG and EB (or at least their locations) would seem to need to be transferred back to the USDF for inclusion in that submission.
If a direct submission to the local batch system is performed, as BPS might normally do, then a global view of the workflow execution is difficult to maintain.

Redis-based QG + EB
-------------------

Today the QG and EB are materialized as files.
For efficiency, it has been proposed to use a Redis database as the persisted (and unified) form of these concepts.
Obviously this requires a Redis server at each site.
But QG generation directly to a remote Redis server seems undesirable, so this implementation might best be paired with site-local QG generation as described above.
Otherwise, the (unified) QG and EB could be transferred as a file (likely via a non-Rucio mechanism) and then loaded into the remote Redis.

PanDA staging
-------------

Today PanDA jobs are not provided with information about the local URIs of the datasets that are to be processed.
This information is contained only in the QG and EB.
But it would be possible to extract that information and provide it to PanDA, enabling it to stage the data from site-local storage to the worker node executing the job rather than having the Butler pull it from site-local storage.
At this level, this is not really related to the multi-site problem.

Given a "Rucio URI"-based source Butler at the USDF, it could also be possible to provide those "Rucio URIs" (DIDs) to PanDA for each job, in which case PanDA could schedule jobs where the data is present in addition to staging.
This seems closer to typical High Energy Physics (HEP) usage.
This has the potential of running afoul of the shared-temporary issue mentioned previously, however.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
