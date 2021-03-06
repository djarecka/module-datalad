---
title: "ReproIn/DataLad: A complete portable and reproducible fMRI study from scratch"
teaching: 5
exercises: 25
questions:
- "How to implement basic neuroimaging study with complete and unambiguous provenance tracking of all actions?"
objectives:
- "Conduct portable and reproducible analyses with ReproIn and DataLad from ground up."
keypoints:
- "TODO"
---
## Introduction

In this lesson we will carry out a full (although very basic) functional
imaging study, going from raw data to complete data analysis results.  We will
start from imaging data that is still in DICOM form, as it would be, if we had
just finished scanning.  Importantly, we will conduct this analysis in a way
that

- leaves a comprehensive "paper-trail" of all performed steps: everything
  will be tracked via version control

- structures components in a way that facilitates re-use

- perform all critical computation in containerized computational environments
  for improved reproducibility and portability

For all steps of this study we will use [DataLad] to achieve these goals with relatively
minimal effort


> #### DataLad extensions
> 
> [DataLad] itself is a data management package that is completely agnostic of
> data formats and analysis procedures. However, DataLad functionality can be
> extended via so-called *extension packages* that add additional support for
> particular data formats, and workflows. In this lesson we will make use
> of two extension packages: [datalad-neuroimaging] and [datalad-container].
>
> {:callout}


## Prepare data for analysis

In order to be able to analyze imaging data we typically have to convert them
from their original DICOM format into NIfTI files. While there are many
programs for this purpose that we could use to create some kind of ad-hoc
directory structure, we can gain a lot by adopting the [BIDS] standard.  There
are many advantages of doing so, but the most important for us is that we have
much less work to do when using BIDS, in comparision to inventing something new.

The data we are working with already follows the [ReproIn] naming conventions,
and the [HeuDiConv] converter can use this information to automatically create
a BIDS-compliant dataset for us.

[![ReproIn Convention](../fig/dbic-conversions.png)](https://github.com/repronim/reproin#overall-workflow)

Our first goal is to convert our DICOM data into a DataLad
dataset in BIDS format.


> ## Task: Create a new DataLad dataset called `bids`
>
> Use the [datalad create] command
>
> > ## Solution
> > ~~~
> > % datalad create bids
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

We now have the new dataset in a directory `bids/`. For all further commands
we will change into this directory to be able to use relative paths.

> ~~~
> % cd bids
> ~~~
> {: .bash}

> #### Advantages of relative path specification
> 
> In many cases it doesn't matter whether one uses absolute or relative paths.
> However, it matters a lot in terms of portability. Using relative paths makes
> it possible to move a dataset to another folder (or give it to a colleague
> using a different computer), and still have all code working.
>
> {:callout}

When using DataLad, it is best to always run scripts from the root directory
of a dataset, and code all scripts using paths that are relative to this
root directory. In order to makes this work, a dataset should contain all
inputs of a processing step (all code, and all data).

That means that we should add the raw DICOM files to our BIDS dataset. In our
case these DICOMs are already available in a DataLad dataset from
[GitHub](https://github.com/datalad/example-dicom-functional.git) that we can
add as a *subdataset* to our BIDS dataset.

> ## Task: Add DICOM data as a subdataset in `inputs/rawdata`
>
> Use the [datalad install] command. Make sure to identify the `bids` dataset
> as the dataset to operate on to get DICOMs registered as a subdataset
> (and not just downloaded as a standalone dataset). Use the [datalad subdatasets]
> command to verify the result.
>
> > ## Solution
> > ~~~
> > % datalad install -d . -s https://github.com/datalad/example-dicom-functional.git inputs/rawdata
> > % datalad subdatasets
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}


At this point we are pretty much ready to convert our DICOMs. However, this is
also a good moment to pause and remember that any software you are using has
the potential to be broken (and probably actually is to some degree). DICOM
conversion is the first transformation that is done to the raw data. If there
is a problem with this step, it can affect all subsequent analysis steps and
the final results. Hence we want to make extra sure that we know exactly what
software we are using and that we can go back to it at a later stage, should we
have the need to investigate and issue.

### Working with containers

Containerized computational environments are a great way to handle this situation.
DataLad (via its [datalad-container] extension) provides support for managing
and using such environments for data processing. This basic idea is to also add
an image of a computational environment to a DataLad dataset (just like any other)
file. This way we know exactly what we are using, and where to get it again
in the future.

A ready-made [singularity] container with the [HeuDiConv] DICOM converter
(~160 MB) is available from [singularity-hub] at:
shub://mih/ohbm2018-training:heudiconv

> ## Task: Add the HeuDiConv container to the dataset
>
> Use the [datalad containers-add] command to add this container under the name
> `heudiconv`, and the [datalad containers-list] command to verify that
> everything worked.
>
> > ## Solution
> > ~~~
> > % datalad containers-add heudiconv -u shub://mih/ohbm2018-training:heudiconv
> > % datalad containers-list
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

We have now reached the point where our dataset trackes all input data, and
the computational environment for the DICOM conversion. This means that we
have a complete record of all components (so far) in this one dataset that
we can reference via relative paths from the dataset root. This is a very good
starting point for conducting a portable analysis.

Datalad comes with two command to run arbitrary shell command and capture there
output in a dataset. These command are [datalad run] and [datalad
containers-run] \(provided by the [datalad-container] extension). Both commands
are very similar, in fact there interface is almost identical. The big
difference is that [datalad run] executes commands in the local environment,
[datalad containers-run] executes commands in a containerized computational
environment. Therefore we are going to use the latter.

> ## Task: Convert DICOM using HeuDiConv from the container
>
> An appropriate [HeuDiConv] command looks like this:
>
>     heudiconv -f reproin -s 02 -c dcm2niix -b -l "" --minmeta -a . \
>         -o /tmp/heudiconv.sub-02 --files inputs/rawdata/dicoms
>
> It essentially tells it to use the [ReproIn heuristic] to convert the
> DICOMs using the subject identifier `02`, with the DICOM converter
> `dcm2niix` into the BIDS format. As the last and most important argument
> the directory with the DICOMs is given.
>
> Run this command through [datalad containers-run], such that the results
> of this command are saved using a meaningful commit message. If something
> goes wrong, remember that you can `git reset --hard` the dataset repository
> to throw away anything that wasn't committed yet.
>
> Once done, use the [datalad diff] command to compare the dataset to the
> previous saved state (`HEAD~1`) to get an instant summary of the changes.
>
> > ## Solution
> > It is sufficient to prefix the original command line call with
> > `datalad containers-run -m "<some message>"`.
> > ~~~
> > % datalad containers-run -m "Convert sub-02 DICOMs into BIDS" \
> >       heudiconv -f reproin -s 02 -c dcm2niix -b -l "" --minmeta -a . \
> >       -o /tmp/heudiconv.sub-02 --files inputs/rawdata/dicoms
> > % datalad diff --revision HEAD~1
> > ~~~
> > {: .bash}
> > It is not necessary to specify the name of the container to be used.
> > If there is only one container known to a dataset [datalad containers-run]
> > is clever enough to use that one.
> {: .solution}
>
{: .challenge}

You can now confirm that a NIfTI file has been added to the dataset, and its
name is compliant with the [BIDS] standard. Information such as the task-label
has been extracted from the imaging sequence description automatically.

There is only one very last thing missing before we can analyze our functional
imaging data: we need to know what stimulation was done at which point during
the scan. Thanksfully the data was collected using an implementation that
experted this information directly in the [BIDS] `events.tsv` format. The
file already came with our DICOM dataset and can be found at
`inputs/rawdata/events.tsv`. The only thing we need to do is to copy it at
the right location under the BIDS-mandated name.

To do that, we could simple use the `cp` shell command. However, in the
history of the dataset it would look like this file came out of nowhere.
Again, we can use DataLad to capture this information. For such a simple
thing like `cp` we do not need a container, so let's use [datalad run]
directly.

> ## Task: Copy the event.tsv file to its correct location via `run`
>
> BIDS requires to put this file at `sub-02/func/sub-02_task-oneback_run-01_events.tsv`.
> Use the [datalad run] command to execute the shell `cp` command to
> implement this step. This time, however, use the options `--input` and
> `--output` to inform [datalad run] what files need tp be available,
> and what locations need to be writeable for this command.
>
> In order to avoid duplication, [datalad run] supports placeholder labels
> that you can use in the command specification itself.
>
> Use `git log` to investigate what information [DataLad] captured about this
> command executions.
>
> > ## Solution
> > It is sufficient to prefix the original command line call with
> > `datalad containers-run -m "<some message>"`.
> > ~~~
> > % datalad run -m "Import stimulation events" \
> >       --input inputs/rawdata/events.tsv \
> >       --output sub-02/func/sub-02_task-oneback_run-01_events.tsv \
> >       cp {{inputs}} {{outputs}}
> > ~~~
> > {: .bash}
> > It is not necessary to specify the name of the container to be used.
> > If there is only one container known to a dataset [datalad containers-run]
> > is clever enough to use that one.
> {: .solution}
>
{: .challenge}

And now we are ready with the data preparation. We have (the skeleton) of a
BIDS-compliant dataset that contains all data in the right format and using the
correct file names. In addition is also tracked the computational environment
used to perform the DICOM conversion, and also tracks a separate dataset
with the input DICOM data. This means we can trace every single file in this
dataset back to its origin, including the commands and inputs used to create
it.

This dataset is now ready. It can be archived, and used as input for one or more
analyses of any kind. Let's leave the dataset directory now:

> ~~~
> % cd ..
> ~~~
> {: .bash}

## A reproducible GLM demo analysis

With our raw data prepared in BIDS format, we can now conduct an analysis.
We will implement a very basic first-level GLM analysis using FSL that runs
in just a few minutes. We will follow the same principles that we already
applied when we prepared the BIDS dataset: complete capture of all inputs,
computational environments and code, as well as outputs.

Importantly, we will conduct our analysis in a new dataset. The raw BIDS
dataset is suitable for many different analysis than can all use that dataset
as input. In order to avoid wasteful duplication and improve the modularity
of our data structures, we will merely use the BIDS dataset as an input,
but we will *not* modify it in any way.

> ## Task: Create a new DataLad dataset called `glm_analysis`
>
> Use the [datalad create] command, and subsequently change into the
> root directory of the newly created dataset.
>
> > ## Solution
> > ~~~
> > % datalad create glm_analysis
> > % cd glm_analysis
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

Following the same logic and commands as previously, we add the raw BIDS
dataset as a subdataset of the new analysis dataset to enable comprehensive
tracking of all input data within the analysis dataset.

> ## Task: Add BIDS data as a subdataset in `inputs/rawdata`
>
> Use the [datalad install] command. Make sure to identify the analysis dataset
> as the dataset to operate on to get the BIDS dataset registered as a subdataset
> (and not just as a standalone dataset). Use the [datalad subdatasets]
> command to verify the result.
>
> > ## Solution
> > ~~~
> > % datalad install -d . -s ../bids inputs/rawdata
> > % datalad subdatasets
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

Regarding the layout of this analysis dataset we cannot relying on automatic
tools and a comprehensive standard yet (but such guidelines are actively being
worked on). However, Datalad nevertheless aids efforts to bring order to the
chaos. Anyone can develop their own ideas on how a dataset should be
structured, and implement these concepts in *dataset procedures* that can be
executed using the [datalad run-procedure] command.

Here we are going to adopt the YODA principles, a set of simple rules on how to
structure analysis dataset. You can learn more about YODA at OHBM poster 2046
(*YODA: YODA’s organigram on data analysis*), but here the only relevant aspect
is that we want to keep all analysis scripts in the subdirectory `code/` of
this dataset. We can get a readily configured dataset by running the YODA
setup procedure:

> ## Task: Run the `setup_yoda_dataset` procedure
>
> Use the [datalad run-procedure] command. Check what has changed in the dataset.
>
> > ## Solution
> > ~~~
> > % datalad run-procedure setup_yoda_dataset
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

Now we are almost ready to fire-up FSL for our GLM analysis. However, we need two
pieces of custom code:

1. a small script that can convert BIDS events.tsv files into the EV3 format that
   FSL can understand: available at <https://raw.githubusercontent.com/myyoda/ohbm2018-training/master/scripts/events2ev3.sh>

2. an FSL analysis configuration template script available at: <https://raw.githubusercontent.com/myyoda/ohbm2018-training/master/scripts/ffa_design.fsf>

Any custom code needs to be tracked, if we want to achieve a complete record of
how an analysis was conducted. Hence we have to store those scripts in our analysis
dataset.

> ## Download the scripts and include them in the analysis dataset
>
> Use the [datalad download-url] command. Place the scripts in the `code/` directory
> under their respective names. Check `git log` to confirm that the commit message
> shows the URL where each script has been downloaded from.
>
> > ## Solution
> > ~~~
> > % datalad download-url -O code/events2ev3.sh https://raw.githubusercontent.com/myyoda/ohbm2018-training/master/scripts/events2ev3.sh
> > % datalad download-url -O code/ffa_design.fsf https://raw.githubusercontent.com/myyoda/ohbm2018-training/master/scripts/ffa_design.fsf
> > % git log
> > 
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

At this point our analysis dataset contains all required inputs. We only have to
run our custom code to produce the inputs in the format that FSL expects.
First, let's convert the events.tsv file into EV3 format files.

> ## Task: Run the converter script for the event timing information
>
> Use the [datalad run] command to execute the script at `code/events2ev3.sh`.
> it requires the name of the output directory (use `sub-02`) and the location
> of the BIDS events.tsv file to convert. Use the `--input` and `--output`
> options to let DataLad automatically manage these files for you.
> **Important**: This BIDS subdataset does not actually have the content for the
> events.tsv file yet. If you use `--input` correctly, DataLad will obtain the
> file content for you automatically. Check the output carefully, the script is
> written in a sloppy way that will produce some output even things go wrong.
> Each generated file must have three numbers per line.
>
> > ## Solution
> > ~~~
> > % datalad run -m 'Build FSL EV3 design files' \
> >       --input inputs/rawdata/sub-02/func/sub-02_task-oneback_run-01_events.tsv \
> >       --output 'sub-02/onsets' \
> >       bash code/events2ev3.sh sub-02 {inputs}
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

And finally we only have left to configure the desired first-level GLM analysis
with FSL. The following command will create a working configuration from the
template we have stored in `code/`. It uses the mighty `sed` editor. That one is
mind-boggling, but once you know how to use it, you never want to forget about
it again. Let's also run that through [datalad run], so we know forever that we didn't
type this in by hand, but actually generated it from a template (that we could
alter and then regenerate this file).

> ~~~
> datalad run \
>     -m "FSL FEAT analysis config script" \
>     --output sub-02/1stlvl_design.fsf \
>     bash -c 'sed -e "s,##BASEPATH##,{pwd},g" -e "s,##SUB##,sub-02,g" \
>         code/ffa_design.fsf > {outputs}'
> ~~~
> {: .bash}

Ready for FSL!

But hold on. We cannot simply run FSL. If we were concerned that a simple DICOM
converter can be buggy, we absolutely have to handle a software as complex as
FSL with the same care. So let's add a container to this analysis dataset too.
A ready-made container with FSL (~260 MB) is available from
shub://mih/ohbm2018-training:fsl

> ## Task: Add a container with FSL
>
> Use the [datalad containers-add] command to add this container under the name
> `fsl`, and the [datalad containers-list] command to verify that
> everything worked.
>
> > ## Solution
> > ~~~
> > % datalad containers-add fsl -u shub://mih/ohbm2018-training:fsl
> > % datalad containers-list
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

And finally, no but, no waiting: We can run FSL. The command is as short as
`feat sub-02/1stlvl_design.fsf`. However, in order to achieve the most reproducible
and most portable execution we should tell the [datalad containers-run] command
what the inputs and outputs are. DataLad will then be able to obtain the required
NIfTI time series file form the BIDS raw subdataset.

Please run the following command as soon as possible, it takes around 5min to
complete on an average system.

> ~~~
> datalad containers-run -m "sub-02 1st-level GLM" \
    --input sub-02/1stlvl_design.fsf \
    --input sub-02/onsets \
    --input inputs/rawdata/sub-02/func/sub-02_task-oneback_run-01_bold.nii.gz \
    --output sub-02/1stlvl_glm.feat \
    feat {inputs[0]}
> ~~~

Once this command finished, DataLad will have captures the entire FSL output,
and the dataset will contain a complete record all the way from the input BIDS
dataset to the GLM results (which, by the way, performed an FFA localization on
a real BOLD imaging dataset, take a look!). The BIDS subdataset in turn has a
complete record of all processing down from the raw DICOMs onwards.

TODO: rerun


## Get ready for the afterlife

And because this record is complete, we can now simply throw away the input BIDS
**subdataset** of our analysis.

> ## Task: Verify that the BIDS subdataset is unmodified and uninstall it
>
> Use the [datalad diff] command to check for modifications of the subdataset,
> and the [datalad uninstall] do delete it.
>
> > ## Solution
> > ~~~
> > % datalad diff --revision HEAD~10 -- inputs
> > % datalad uninstall -d . inputs -r
> > ~~~
> > {: .bash}
> {: .solution}
>
{: .challenge}

TODO metadata

[datalad add-sibling]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-add-sibling.html
[datalad add]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-add.html
[datalad annotate-paths]: http://docs.datalad.org/en/latest/generated/man/datalad-annotate-paths.html
[datalad clean]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-clean.html
[datalad clone]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-clone.html
[datalad copy_to]: http://docs.datalad.org/en/latest/_modules/datalad/support/annexrepo.html?highlight=%22copy_to%22
[datalad create-sibling-github]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-create-sibling-github.html
[datalad create-sibling]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-create-sibling.html
[datalad create]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-create.html
[datalad download-url]: http://docs.datalad.org/en/latest/generated/man/datalad-download-url.html
[datalad diff]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-diff.html
[datalad drop]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-drop.html
[datalad export]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-export.html
[datalad export_tarball]: http://docs.datalad.org/en/latest/generated/datalad.plugin.export_tarball.html
[datalad get]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-get.html
[datalad install]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-install.html
[datalad ls]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-ls.html
[datalad metadata]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-metadata.html
[datalad plugin]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-plugin.html
[datalad publish]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-publish.html
[datalad remove]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-remove.html
[datalad run]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-run.html
[datalad run-procedure]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-run-procedure.html
[datalad rerun]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-rerun.html
[datalad save]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-save.html
[datalad search]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-search.html
[datalad siblings]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-siblings.html
[datalad sshrun]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-sshrun.html
[datalad subdatasets]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-subdatasets.html
[datalad update]: http://datalad.readthedocs.io/en/latest/generated/man/datalad-update.html
[datalad containers-add]: http://docs.datalad.org/projects/container/en/latest/generated/man/datalad-containers-add.html
[datalad containers-list]: http://docs.datalad.org/projects/container/en/latest/generated/man/datalad-containers-list.html
[datalad containers-run]: http://docs.datalad.org/projects/container/en/latest/generated/man/datalad-containers-run.html

[ReproIn]: http://reproin.repronim.org
[ReproIn heuristic]: https://github.com/nipy/heudiconv/blob/master/heudiconv/heuristics/reproin.py
[DataLad]: http://datalad.org
[datalad-neuroimaging]: https://github.com/datalad/datalad-neuroimaging
[datalad-container]: https://github.com/datalad/datalad-container
[DataLad]: http://datalad.org
[HeuDiConv]: http://github.com/nipy/heudiconv
[BIDS]: http://bids.neuroimaging.io
[singularity-hub]: https://singularity-hub.org/
[singularity]: http://singularity.lbl.gov/
