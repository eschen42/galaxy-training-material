---
layout: tutorial_hands_on

title: MaxQuant Phopsphoenrichment Protocol Data Processing Pipeline
zenodo_link: ''
questions:
- Which phosphopeptides exhibit a differential treatment effect (ANOVA)?
- Which kinases' substrates are overrepresented among differentially detected phosphopeptides (KSEA)?
objectives:
- Classify phosphopeptides regarding significance of their differential expression.
- Characterize overrepresentation of differentially-phosphorylated substrates observed for each kinase.
time_estimation: 3H
key_points:
- The tools automate the analysis of differential phosphorylation and kinase-substrate enrichment analysis.
- A rudimentary knowledge of regular expressions is required to set some parameters.
tags: [DIA, human]
subtopic: id-quant
contributors:
- eschen42

---

# Introduction

Phosphopeptides constitute a minority of the peptides prepared by proteolysis for shotgun MS analysis.
Enrichment for phosphopeptides simplifies differential phosphorylation analysis ({% cite Cheng_2018 %}).
This data-processing pipeline accepts LC-MS data collected for samples that have been enriched in phosphopeptides as described in Cheng *et al.*.

MaxQuant ({% cite Cox_2008 %}; {% cite Cox_2014 %}) can be used effectively to perform label-free
identification and quantitation phopsphopeptides. However, additional data-processing steps are necessary to analyze these results and provide a context for their interpretation.

This tutorial introduces:

1. Obtaining the prerequisite static files for this pipeline.
1. Running MaxQuant within an subworkflow to create a customized `mqpar.xml` file.
1. Running MaxQuant with a customized `mqpar.xml` file to create the `Phospho (STY)Sites.txt` file.
1. Applying preprocessing to the phosphopeptide sites identified and quantitated by MaxQuant.
1. Applying statistical analysis to the preprocessed data.

This tutorial **focuses on human phosphoproteomics**. However, it might be feasible to use different static files to investigate phosphopeptides other organisms.

This tutorial also **assumes that the LC-MS data are collected in Thermo-Fisher RAW format**.  The workflow will need to be adapted appropriately to accept other data formats.

<!--
[tutorial to learn how to fill the Markdown]({{ site.baseurl }}/topics/contributing/tutorials/create-new-tutorial-content/tutorial.html)**
-->

> <agenda-title></agenda-title>
>
> In this tutorial, we will cover:
>
> 1. TOC
> {:toc}
>
{: .agenda}

# Overview

The data processing of Thermo-Fisher RAW LC-MS data files to ANOVA and KSEA results is summarized in the following figure:

<figure id="figure-1">
  <img
    width="700"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphoenrichment/overview_flowchart.svg"
    alt="The MaxQuant Phosphoenrichment Data Processing Pipeline comprises these steps: construction of `mqpar.xml`; running of MaxQuant; extraction of the `Phospho (STY)Sites.txt` file; preprocessing; statistical analysis with ANOVA and KSEA."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 1:</span> Overview of the MaxQuant Phosphoenrichment Data Processing Pipeline
  </figcaption>
</figure>

This process will be presented in this tutorial as a sequence of several phases:

- **Phase 1.** Gathering (or making) the (several) static input files required.
- **Phase 2.** Uploading the Thermo RAW LC-MS data files and a tabular file describing the samples.
- **Phase 3.** Running MaxQuant, in two steps:
  - Synthesizing the MaxQuant `mqpar.xml` parameters file using the Galaxy [**MaxQuant**](https://toolshed.g2.bx.psu.edu/view/galaxyp/maxquant/4d17fcdeb8ff) tool (e.g., see the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=0c6018b0e3966b99&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F000%2Frepo_410%2Fmaxquant.xml&changeset_revision=837224ad1694&render_repository_actions_for=tool_shed)).
  - Running MaxQuant from `mqpar.xml` using the Galaxy [**MaxQuant (using mqpar.xml)**](https://toolshed.g2.bx.psu.edu/view/galaxyp/maxquant_mqpar/9cb7dcc07dae) tool {e.g., see the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=0c6018b0e3966b99&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F000%2Frepo_410%2Fmaxquant_mqpar.xml&changeset_revision=837224ad1694&render_repository_actions_for=tool_shed)).
- **Phase 4.** Applying preprocessing (phosphosite-localization and protein identification) to the sites identified by MaxQuant using the Galaxy [**MaxQuant Phosphopeptide Preprocessing**](https://toolshed.g2.bx.psu.edu/view/galaxyp/mqppep_preproc/5c2beb4eb41c) tool (e.g., see the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=7b866682d0b1d44c&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6289%2Fmqppep_preproc.xml&changeset_revision=b889e05ce77d&render_repository_actions_for=tool_shed)).
- **Phase 5.** Performing normalization, imputation, ANOVA and KSEA (Kinase-Substrate Enrichment Analysis) using the Galaxy [**MaxQuant Phosphopeptide ANOVA**](https://toolshed.g2.bx.psu.edu/view/galaxyp/mqppep_anova/2d9f216c1048) tool (e.g., see the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=fd3fde90cd863d18&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6288%2Fmqppep_anova.xml&changeset_revision=514afed1f40d&render_repository_actions_for=tool_shed)).

## Four Related Workflows

Four closely related workflows implementing phases 3-5 of phosphoproteomic enrichment data processing workflow are presented in this tutorial, in varying degrees of detail:

You can download [**the "Overview" workflow**]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphoproteomic_Enrichment_Pipeline_-_Overview.ga); this workflow includes of all the steps in the "Quant" and "Postprocessing" workflows described below.  This tutorial discusses the "Quant" and "Postprocessing" workflows instead of discussing the "Overview", but it may be convenient to run the "Overview" workflow once you are familiar with how the other two behave.  Figure 1 at [the beginning of this Overview section](#overview) depicts the "Overview" workflow.

You can download [**the "Quant" workflow**]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphoproteomic_Enrichment_Pipeline_-_Quant.ga); this performs the initial stages (*phase 3*) of the pipeline: Prepare mqpar.xml and run MaxQuant.

<figure id="figure-2">
  <img
    height="250"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphoenrichment/overview_flowchart_phase3.svg"
    alt="The Quant Phase of the MaxQuant Phosphoenrichment Data Processing Pipeline comprises these steps: construction of `mqpar.xml`; running of MaxQuant; extraction of the `Phospho (STY)Sites.txt` file."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 2:</span> The "Quant" phase of the pipeline
  </figcaption>
</figure>

You can download [**the "Postprocessing" workflow**]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphoproteomic_Enrichment_Pipeline_-_Postprocessing.ga); this performs the post-MaxQuant half (*phases 4 and 5*) of the pipeline: Preprocess results and produce ANOVA/KSEA report.

<figure id="figure-3">
  <img
    height="250"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphoenrichment/overview_flowchart_phases4and5.svg"
    alt="The Postprocessing Phases of the MaxQuant Phosphoenrichment Data Processing Pipeline comprises these steps: preprocessing; statistical analysis with ANOVA and KSEA."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 3:</span> `mqpar.xml` Postprocessing phases of the pipeline
  </figcaption>
</figure>

You can download [**the "Synthesize mqpar.xml" subworkflow**]({{ site.baseurl }}{{ page.dir }}workflows/subworkflow-synthesize_mqpar.xml_for_MaxQuant.ga); this subworkflow is invoked by the "Quant" workflow, produces a list of mzXML and fully-specified `mqpar.xml` for MaxQuant run.  A copy of this subworkflow is included and annotated here for inspection, but it will not be discussed further in this tutorial.

<figure id="figure-4">
  <img
    height="250"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphoenrichment/overview_flowchart_phase3a.svg"
    alt="The mqpar.xml contruction subworkflow from the MaxQuant Phosphoenrichment Data Processing Pipeline."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 4:</span> The `mqpar.xml` construction subworkflow of "Quant" phase
  </figcaption>
</figure>

## Static inputs

**Several files will change little if at all from one run of the pipeline to the next.**  You will probably get them (directly or from a colleague) and put them into a Galaxy history from which you can copy them when starting a new run of the pipeline.

1. **FASTA files**
  - One or more FASTA files bearing the sequences of peptides that you expect to find; you may want files that cover all isoforms.
  - One or more FASTA files bearing the sequences of contaminants that you expect may be present, as described below in [the "FASTA files" section](#fasta-files).
1. An `mqpar.xml` **file generated by MaxQuant with most of the general settings that you require;** e.g., you can save this from the MaxQuant desktop program.
1. A **file of alpha levels** to be used as thresholds in ANOVA analysis.
1. The **four phosphosite databases:**
  - PSP Kinase-Substrate sites
  - PSP Regulatory sites
  - HPRD pSTY motifs
  - NetworKIN sites

## Run-specific inputs

For each run of MaxQuant, you will need to **upload the Thermo RAW LC-MS data files**.

Also, you will need to **create the "Name-to-Fraction-Experiment Map"** described below.

- This file is a simplification of the MaxQuant "Experimental Design" file.

## Synthesizing `mqpar.xml`

Many MaxQuant parameters that do not change from one run to the next.
**Save the invariant MaxQuant parameters** in your `mqpar.xml` parameters template file that you use as input for this step; see ["Static input files"](#static-input-files) above.

**The synthesis step creates a MaxQuant `mqpar.xml` parameters file customized for a specific MaxQuant run**:

- An individual run of the MaxQuant tool also needs specification of the FASTA files, data files, and an experimental design file for that run.
- The first part of the MaxQuant workflow presented here "splices" the invariant parts of `mqpar.xml` with these paths.

## Running MaxQuant

The next phase is to run MaxQuant to **identify and quantitate the phosphopeptides**.

The actual running of MaxQuant with parameters provided in the `mqpar.xml` takes a very long time, which is why it is better to start with an `mqpar.xml` file in which you have confidence so that runs will be reliable and reproducible.  Among the outputs of MaxQuant will be the `Phospho (STY)Sites.txt` file that will be used by the next phase.

## Preprocessing MQ results

The "preprocessing" phase accepts the `Phospho (STY)Sites.txt` file from the previous phase and performs three steps:

1. Applying a user-specfied "localization probability" threshold (`0.75` by default) to **select phopshopeptides for which there is sufficient confidence in the location(s) of phosphorylation**.
2. "Mapping upstream kinases", i.e., **match the phosphopeptides to four databases** of kinases recognition and phosphorylation sites reported in the literature: PhosphoSite ({% cite Hornbeck_2004 %}, {% cite Hornbeck_2011 %} ), PhosphoMotif Finder ({% cite Amanchy_2007 %}), Phosida ({% cite Gnad_2010 %}), and NetworKIN ({% cite Linding_2007 %}, {% cite Linding_2007a %}a).
  - As a prerequisite to mapping, this step must **search the FASTA file(s) of expected protein sequences for matches to the phosphopeptides** detected, characterized, and quantitated by MaxQuant.
3. **Merging** the database search results with the selected phosphopeptides, **optionally filtering the database results by species**.
  - This step produces outputs tailored to the statistical analysis step

For further info, please see the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=7b866682d0b1d44c&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6289%2Fmqppep_preproc.xml&changeset_revision=b889e05ce77d&render_repository_actions_for=tool_shed).

## ANOVA and KSEA

The final step covered here is performing a pair of statistical analyses after some preparatory steps:

- Parse sample names with regular expressions to **assign samples to treatment groups**.
- Perform **quantile normalization** ({% cite Bolstad_2003 %}).
- Perform **imputation of missing values** by one of four possible methods:
  - Insert a small random value.
  - Insert the mean of non-missing values.
  - Insert the median of non-missing values.
  - Insert the median of non-missing values for samples in the same treatment group.
- Compute **rank** (and possibly filter) **phosphopeptides on the basis of a "quality score"** that is assigned to each substrate, as described in [the "Statistical notes" section below](#statistical-notes).
  - This score takes into account both FDR-adjusted p-value and the number of missing values for each substrate.
- Perform **ANOVA** (analysis of variance), which is a **univariate** technique used to identify **each phosphopeptide for which there is considerable evidence against the null hypothesis** that their variation among the several treatments occurs by random chance.  There are several challenges to interpreting these data:
  - The data reflect the abundance of particular protein modifications, which may result in relative differences in activity, but they do not give any sense regarding the regulatory signals that led to these differences.
  - Inter-sample, intra-treatment variation for individual phosphopeptides can be extensive (indeed, missing values are common).  **A small treatment effect signal can be swamped by the "noise".**
- Perform **KSEA** (Kinase-Substrate Enrichment Analysis), which groups by kinase the subtle differences among the individual phosphopeptide substrates and **estimates the likelihood that the substrates of each kinase differ more than would be expected at random**.
  - In effect, even though it may be likely that individual substrates' differences might be expected because of random variation, the likelihood of the aggregation of substrates of a given kinase differing by random chance as much as is observed may be rather unlikely.
  - The KSEA algorithm used here is from {% cite Casado_2013 %} as implemented in the KSEAapp package reported in {% cite Wiredja_2017 %}.  The code used here is adapted from the CRAN R package KSEAapp ({% cite Wiredja_2017a %}a) to work with output from the "MaxQuant Phosphopeptide Preprocessing" Galaxy tool and the multiple kinase-substrate databases that the latter tool searches..
- Produce as **outputs**:
  - A PDF report for all of these steps. 
  - Datasets with the imputed phosphopeptide intensities.
  - Datasets with the ANOVA and KSEA results for *ad hoc* analysis or report creation.

<figure id="figure-5">
  <img
    width="700"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphoenrichment/KSEA_impl_flowchart.svg"
    alt="Flowchart showing steps in statistical analysis with ANOVA and KSEA."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 5:</span> ANOVA and KSEA processing steps
  </figcaption>
</figure>

For details, please see the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=fd3fde90cd863d18&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6288%2Fmqppep_anova.xml&changeset_revision=514afed1f40d&render_repository_actions_for=tool_shed).

# Static inputs

## FASTA files

You will want to have:

- one or more **FASTA files containing expected peptide sequences** for the MaxQuant step and for subsequent peptide/protein searches during the preprocessing step.
  - You may want an "all isozymes" version because your experimental data likely include phosphopeptides from all isozymes, not just the "most representative isozyme".
- zero or more **FASTA files containing expected "contaminating" peptide sequences** for the MaxQuant step.
  - For tissue culture, you may wish to add proteomes of *likely* *Mycoplasma* contaminants, by either of the methods described above, decorating with the "CONTAMINANT" flags as described in [the "Contaminant databases" section of the Protein FASTA Database Handling tutorial](https://training.galaxyproject.org/training-material/topics/proteomics/tutorials/database-handling/tutorial.html#contaminant-databases).
    - For an explanation of the considerations to make, see the "Custom Contaminant Detection (Mycoplasma)" section of ({% cite Bielow_2016 %}).
  - For chimeric tissue samples such as human tissue explanted into mouse tissue, you will want the "all isozymes" proteome of the organism that is not the subject of the experiment.
- Note that the decisions made here will influence the PTXQC report produced by the step in which MaxQuant runs.

> <details-title>Retrieving FASTA for all human isoforms from UniProt</details-title>
>
> Follow the process described at
> [https://www.uniprot.org/help/retrieve\_sets](https://www.uniprot.org/help/retrieve_sets)
> to run some scripts to download all isoforms of a proteome.
>
> For example, to download all isoforms of the human proteome:
>
> - Using a web browser:
>   - Navigate to [https://www.uniprot.org/uniprotkb?query=proteome:UP000005640](https://www.uniprot.org/uniprotkb?query=proteome:UP000005640)
>   - Click the `Download` button (which looks like this &nbsp; <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32" fill="blue" width="14" height="14"><path d="M27.6 21h3.9v7.7H.5v-7.8h3.9v4h23.2v-4zm-3.9-9.7L16 22.9 8.2 11.3h5.9V1.6h3.8v9.7h5.9z"></path></svg>)
>   - Choose:
>     - `Download all`
>     - Format `FASTA (canonical & isoform)`
>     - Compressed: `yes`
>   - Click download, and a "gzipped" FASTA file will download.
>   - From the command line, uncompress the download with the `gzip -d` command.
>     - Alternatively, you may use a multifunctional decompression program like 7-zip ([https://www.7-zip.org/](https://www.7-zip.org/)).
> - Alternatively, if you know the proteome ID, you can use "Paste/Fetch data" with a URL like this for human proteome UP000005640:
>   ```
https://rest.uniprot.org/uniprotkb/stream?format=fasta&includeIsoform=true&query=(proteome:UP000005640)
```
> {% snippet faqs/galaxy/datasets_import_via_link.md %}
>
{: .details}

> <details-title>Retrieving FASTA for all mouse isoforms from UniProt</details-title>
>
> You can use "Paste/Fetch data" with a URL like this for mouse proteome UP000000589:
>   ```
https://rest.uniprot.org/uniprotkb/stream?format=fasta&includeIsoform=true&query=(proteome:UP000000589)
```
>
{: .details}

## MaxQuant `mqpar.xml` parameters file

The **mqpar.xml** file has myriad parameters for the many steps in the sophisticated pipeline that the MaxQuant tool implements.  A great place to start familiarizing yourself with these parameters is in [the **Label-free data analysis using MaxQuant** tutorial]({{ site.baseurl }}/topics/proteomics/tutorials/maxquant-label-free/tutorial.html#maxquant-analysis).

Although this file does include paths to your FASTA files, your LC-MS data files, and your experimental design details, it also contains many parameters that you will never change from experiment to experiment.  This is why you want to set up a reliable starter `mqpar.xml` file; assignment of the FASTA files, data files, and experimental design are part of the first MaxQuant step, so don't worry about them (or bother putting them into the MaxQuant GUI) at this time.

Ideally, you would get a starter `mqpar.xml` from a colleague who is doing something very similar to what you are doing.  However, if you need to make a lot of changes and want the MaxQuant GUI to guide you (which is recommended), you can use
```
  File -> Load Parameters ..
```
to load your starter `mqpar.xml` file (if you have one).

After you have customized your settings, you can use
```
  File -> Save Parameters ..
```
to save the modified `mqpar.xml` file.

> <details-title>An example "starter <code>mqpar.xml</code> file"</details-title>
>
> Here is [a starter `mqpar.xml` file](./faqs/mqpar.xml) that could possibly have some or most of the settings that you need.
>
{: .details}

## Databases for preprocessing

Four tabular phosphosite-database files are required for the preprocessing step.

  - `networkin_cutoff_2.tabular`
  - `pSTY_motifs.tabular`
  - `psp_kinase_substrate_dataset.tabular`
  - `psp_regulatory_sites.tabular`
  - ("vaporware alert": Future editions of the tools may provide the options of excluding databases or adding additional databases.)

These databases are, to varying degrees, covered by licenses designed to limit them to academic or non-commercial use.
*If you can comply with the licenses* but cannot obtain the site databases from a colleague, you can:

> <details-title>Create the four phosphosite databases from scratch</details-title>
>
> Follow the process described at
> [https://github.com/eschen42/combine\_phospho\_dbs#Purpose](https://github.com/eschen42/combine_phospho_dbs#Purpose)
> to run some scripts to download and assemble these files.
>
> - There might be no option to DOI assembled files because of licenses restricting them to non-commercial use.
>   - You will review and agree to the license stipulations as part of the procedure.
>   - Once you have agreed to the licenses and followed the procedure, the scripts can run.
> - The scripts run under Linux, where all of the dependencies may be met easily through Conda.
>   - Some additional work will be necessary to run them under Windows or MacOS; in such cases, it may be easiest to run them in a Docker container.
>
{: .details}

## Alpha-thresholds file

The ANOVA step filters the results that it presents based on an alpha-threshold, which is the adjusted ANOVA *p*-value for a phosphopeptide to be included in the list of significant results.  These threshold values have no impact on the results from KSEA.

This file is very simple: it is a headerless file with one alpha-threshold per line.  Technically, it is a tabular file but has no tabs!  For example, it may look like this:
```
0.01
0.05
0.1
```

## Save your static files

Initially, you can **create a history to hold your files that rarely change.**  Indeed, you might be able to copy a history published by a colleague on your Galaxy instance.

> <hands-on-title> Upload static inputs </hands-on-title>
>
> 1. Create **a new history** for this tutorial and name it `mqppep tutorial part 1 - Static inputs`.
> 2. Upload **a "starter" `mqpar.xml` dataset**:
>   - Here is [a starter `mqpar.xml` file](./faqs/mqpar.xml) that we will use for this tutorial.
> 3. Add your **alpha-thresholds dataset** by pasting
>   - Click the "Upload Data" button at the top of the left pane, then type in the values.
>
>    {% snippet faqs/galaxy/datasets_create_new_file.md %}
>
>    For further discussion of the "upload by paste" process, please see
>    [the two "paste datasets via URL" slides beginning here]({{ site.baseurl }}/topics/galaxy-interface/tutorials/get-data/slides.html#23),
> 4. Import **the four phosphosite databases**:
>   - Ordinarily, you would obtain these as described in the section ["Databases for preprocessing"](#databases-for-preprocessing) above.
>   - For this tutorial, however, we will use some *small subsets* of these databases.
>     Click the "Upload Data" button at the top of the left pane, then upload (using "Paste/Fetch data") these databases as datatype "tabular" using the following URLs (being careful to have no leading spaces):
> ```
> https://raw.githubusercontent.com/galaxyproteomics/tools-galaxyp/master/tools/mqppep/test-data/pSTY_motifs.tabular
> https://raw.githubusercontent.com/galaxyproteomics/tools-galaxyp/master/tools/mqppep/test-data/test_networkin.tabular
> https://raw.githubusercontent.com/galaxyproteomics/tools-galaxyp/master/tools/mqppep/test-data/test_kinase_substrate.tabular
> https://raw.githubusercontent.com/galaxyproteomics/tools-galaxyp/master/tools/mqppep/test-data/test_regulatory_sites.tabular
> ```
>    {% snippet faqs/galaxy/datasets_import_via_link.md %}
>
>    For further discussion, please see
>    [the two "Directly enter text" slides beginning here]({{ site.baseurl }}/topics/galaxy-interface/tutorials/get-data/slides.html#26),
> 1. Obtain the **FASTA file for protein sequences (all isoforms) expected** for the species under study
>   - For this tutorial, however, we will use all human isoforms from a human proteome (UP000005640).
>     Upload (using "Paste/Fetch data") this databases as datatype "fasta" using the following URL:
> ```
> https://rest.uniprot.org/uniprotkb/stream?format=fasta&includeIsoform=true&query=(proteome:UP000005640)
> ```
>     Depending on your connection speed, this may take three minutes or more.
>   - Alternatively, you can use the [**Protein Database Downloader** tool](https://toolshed.g2.bx.psu.edu/view/galaxyp/dbbuilder/983bf725dfc2), choosing:
>
>     - `Download from: UniProtKB`
>     - `Taxonomy: Homo sapiens (Human)`
>     - `taxon_id: (leave blank)`
>     - `Reviewed: UniProtKB`
>     - `Proteome Set: Reference Proteome Set`
>     - `Include isoform data: Yes`
>
>     See further discussion in the
>     ["Uploading a protein database"]({{ site.baseurl }}/topics/database-handling/tutorial.html#uploading-a-protein-search-database)
>     section of the "Protein FASTA Database Handling" GTN.
> 1. Obtain one or more FASTA files for contaminant protein sequences.
>   - For this tutorial, however, **we will use the "common Repository of Adventitious Proteins"**; obtain this as described in the
>     ["Contaminant databases"]({{ site.baseurl }}/topics/database-handling/tutorial.html#contaminant-databases)
>     section of the "Protein FASTA Database Handling" GTN using the [**Protein Database Downloader** tool](https://toolshed.g2.bx.psu.edu/view/galaxyp/dbbuilder/983bf725dfc2), choosing:
>
>     - `Download from: cRAP (contaminants)`
>
>     Alternatively, you can paste-fetch from:
> ```
> https://raw.githubusercontent.com/pravs3683/cRAP/master/cRAP_protein_database.fasta
> ```
>
>
> Notes:
>
> 1. You can fix any incorrectly datatyped datasets.
>
>    {% snippet faqs/galaxy/datasets_change_datatype.md datatype="datatypes" %}
>
> 1. If desired, add to each dataset a tag if you want the tag to propagate to the outputs.
>
>    {% snippet faqs/galaxy/datasets_add_tag.md %}
>
{: .hands_on}

> <question-title></question-title>
>
> What datasets should you have in your `mqppep tutorial part 1 - Static inputs` history now?
>
> > <solution-title></solution-title>
> >
> > - `mqpar.xml`
> > - `alpha-thresholds`
> > - `pSTY_motifs.tabular`
> > - `test_networkin.tabular`
> > - `test_kinase_substrate.tabular`
> > - `test_regulatory_sites.tabular`
> > - `HumanProteome_UP000005640_AllIsozymes.fasta`
> > - `common Repository of Adventitious Proteins.fasta`
> >
> {: .solution}
{: .question}

# Run-specific inputs

## Experimental data history

> <hands-on-title> Upload LC-MS data </hands-on-title>
>
> - Create **a new history** named `mqppep tutorial part 2 - all inputs`.
>
> - "Paste/Fetch" or upload the **Thermo RAW LC-MS data files** to the new history.
>
> {% snippet faqs/galaxy/datasets_upload.md %}
>
>
> For this tutorial, please Paste/Fetch the following four spectra from {% cite VanDeusen_2020 %}, each as datatype `thermo.raw`:
>
> ```
> https://www.ebi.ac.uk/pride/data/archive/2020/05/PXD012971/QE03659.raw
> https://www.ebi.ac.uk/pride/data/archive/2020/05/PXD012971/QE03659-2.raw
> https://www.ebi.ac.uk/pride/data/archive/2020/05/PXD012971/QE03670.raw
> https://www.ebi.ac.uk/pride/data/archive/2020/05/PXD012971/QE03670-2.raw
> ```
>
> The two `QE03659` samples are replicated from an androgen-receptor dependent cell line (AR), and the two `QE03670` samples are replicated from an androgen-receptor dependent "Aggressive Variant Prostate Cancer" cell line (AVPC).
{: .hands_on}

> <hands-on-title> Create a dataset collection with the LC-MS data </hands-on-title>
>
> - Build **a "dataset collection" containing the Thermo RAW datasets**, naming it `pST Thermo RAW`.
>   - **Be sure that you only include Thermo RAW datasets.**  Otherwise, the collection will not work properly.
>   - Unless you uncheck `Hide original elements?`, your Thermo RAW datasets will be hidden.
>
> {% snippet faqs/galaxy/collections_build_list.md %}
{: .hands_on}


## "Name-to-Fraction-Experiment Map"

To create "experimental design" input for MaxQuant, the workflow uses a tabular file mapping sample name to fraction and experiment.  It must have three named columns: `Name`, `Fraction`, and `Experiment`.

1. The first column, `Name`, should have names *exactly* matching the Thermo RAW file names, minus any .raw or .thermo.raw file extensions.
2. The second column, `Fraction`, distinguishes fractions among the same sample.
3. The third column (which will be the same for all fractions of one sample), `Experiment`, affects the labeling of output columns.
  - You need to encode the treatment and replica into the value in this column so that you can parse it with the `Sample-name extraction pattern` and `Sample-group extraction pattern` parameters passed to the ANOVA/KSEA tool.

> <hands-on-title> Create a name-to-fraction-experiment map dataset </hands-on-title>
> Upload (by "Paste/Fetch") **a tabular file mapping sample name to fraction and experiment**, named `name2fracexpt_for_pST.tabular`.  In this example, the treatment is coded as `AR` or `AVPC` before the final dot in the experiment label.
>
> > | Name        | Fraction | Experiment         |
> > | ------------------------------------------- |
> > | `QE03659-2` | `1`      | `22Rv1.AR.3B`      |
> > | `QE03659`   | `1`      | `22Rv1.AR.3A`      |
> > | `QE03670-2` | `1`      | `PARCB-5.AVPC.14B` |
> > | `QE03670`   | `1`      | `PARCB-5.AVPC.14A` |
> {: .matrix}
> {% snippet faqs/galaxy/datasets_create_new_file.md %}
{: .hands_on}


> <question-title></question-title>
>
> What datasets should you have in your `mqppep tutorial part 2 - all inputs` history now?
>
> > <solution-title></solution-title>
> >
> > - `QE03659.raw` (may be hidden)
> > - `QE03659-2.raw` (may be hidden)
> > - `QE03679.raw` (may be hidden)
> > - `QE03679-2.raw` (may be hidden)
> > - `pST Thermo RAW` (a dataset collection containing the four files immediately above)
> > - `name2fracexpt_for_pST.tabular`
> >
> {: .solution}
{: .question}

## Copies of static inputs

Copy the static inputs (from section ["save your static files"](#save-your-static-files) above) into the history where you uploaded the data files and the map.

# MaxQuant workflow

## Option 1: "Overview" complete workflow

One option you have is to run the entire workflow (MaxQuant, preprocessing, and statistical analysis) in a single workflow run, the "Overview" workflow.  The "Overview" workflow comprises the steps from the "Quant" and "Postprocessing" partial workflows.  Eventually, you may become familiar enough with the process to adapt and use this workflow, but for the sake of explaining the process, we will cover the second option in the rest of this tutorial.

## Option 2 (part A): "Quant" partial workflow

A second option is to run the "Quant" partial workflow and postpone running the "Postprocessing" partial workflow.  To simplify the presentation, this option is the one described in this tutorial.

> <hands-on-title>Quant (phase 3)</hands-on-title>
>
> 1. **Import the workflow** into Galaxy
>    - Copy (e.g. via right-click) [the URL of the "Quant" workflow]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphoproteomic_Enrichment_Pipeline_-_Quant.ga) and download it to your computer.
>    - Import the workflow into Galaxy
>
>    {% snippet faqs/galaxy/workflows_import.md %}
>
> 2. **Set up the workflow invocation**
>    > {% snippet faqs/galaxy/workflows_run.md %}
>
>    Enter the following parameters:
>
>    > | Input                                                   | Value                                             |
>    > | ------------------------------------------------------- | ------------------------------------------------- |
>    > | `Send results to a new history`                         | `Yes`                                             |
>    > | `History name`                                          | `mqppep tutorial run`                             |
>    > | {% icon param-file %} `FASTA valid phosphopeptides`     | `HumanProteome_UP000005640_AllIsozymes.fasta`     |
>    > | {% icon param-file %} `FASTA contaminants`              | `common Repository of Adventitious Proteins.fasta`|
>    > | `[PARAM] FASTA id parse rule`                           | `>([^\s]*)`                                       |
>    > | `[PARAM] FASTA desc parse rule`                         | `>(*)`                                            |
>    > | {% icon param-file %} `Name-to-Fraction-Experiment Map` | `name2fracexpt_for_pST.tabular`                   |
>    > | {% icon param-file %} `Thermo RAW collection`           | `pST Thermo RAW`                                  |
>    > | {% icon param-file %} `General mqpar.xml`               | `mqpar.xml`                                       |
>    {: .matrix}
>
>    - The two "parse rule" PARAM inputs are specified as PERL-compatible Regular Expressions; see [https://rdrr.io/r/base/regex.html](https://rdrr.io/r/base/regex.html) for help.  Unfortunately, there is a bit of a "learning curve" for regular expressions; patience will be rewarded.
>
> 3. **Click the `Run Workflow` button.**
>
> 4. Wait a *long* time.  (Perhaps surprisingly) you can monitor progress of MaxQuant by looking at the contents of the (running) dataset:
>    - {% icon param-file %} `VIEW PROGRESS HERE`.
>
{: .hands_on}

The output datasets of immediate interest are:
- {% icon param-file %} `phospho_STY_sites.txt`, which  you will take to the Postprocessing workflow.
- {% icon param-file %} `PTXQC_report`; to determine how quality considerations from this report should influence your subsequent analysis, study {% cite Bielow_2016 %} carefully; FYI, this has nothing to do with the "quality score" computed for KSEA.

# Postprocessing workflow

## Option 2 (part B): "Postprocessing" workflow

After you have run the Quant workflow, you will have the {% icon param-file %} `phospho_STY_sites.txt` dataset in hand.  Invocation of the "Postprocessing" workflow initiates a pipeline invoking two substeps:

- Filtering of phosphosite localization probabilities, imputation of missing values, and normalization.
- Statistical analysis with ANOVA and KSEA.

After the pipeline has run, each of these substeps may be re-run with changed parameters, if desired.

> <hands-on-title>Quant (phases 4 and 5)</hands-on-title>
>
> 1. **Import the workflow** into Galaxy
>    - Copy (e.g. via right-click) [the URL of the "Postprocessing" workflow]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphoproteomic_Enrichment_Pipeline_-_Postprocessing.ga) and download it to your computer.
>    - Import the workflow into Galaxy
>
> 2. **Set up the workflow invocation**
>    - **If you wish to change any of the tool parameters**, you can click `Expand to full workflow form.` at the bottom of the form and then expand the two tool steps to reveal the parameters.
>    - The parameters for `MaxQuant Phosphopeptide Preprocessing` are described in the tool help, which you may also see in the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=7b866682d0b1d44c&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6289%2Fmqppep_preproc.xml&changeset_revision=b889e05ce77d&render_repository_actions_for=tool_shed).
>    - The parameters for `MaxQuant Phosphopeptide ANOVA` (which also does KSEA) are described in the tool help, which you may also see in the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=fd3fde90cd863d18&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6288%2Fmqppep_anova.xml&changeset_revision=514afed1f40d&render_repository_actions_for=tool_shed).
>
> 3. **Enter the parameters**
>    > | Input                                                   | Value                                                       |
>    > | ------------------------------------------------------- | ----------------------------------------------------------- |
>    > | `[PARAM] phosphoenrichment (pST or pY)`                 | `pST` or `pY`, depending on the enrichment performed        |
>    > | {% icon param-file %} `FASTA valid phosphopeptides`     | `HumanProteome_UP000005640_AllIsozymes.fasta`               |
>    > | {% icon param-file %} `phospho_STY_sites.tabular`       | {% icon param-file %} `phospho_STY_sites.txt` from "Quant"  |
>    > | {% icon param-file %} `NetworKIN`                       | `test_networkin.tabular`                                    |
>    > | {% icon param-file %} `pSTY_Motifs`                     | `pSTY_motifs.tabular`                                       |
>    > | {% icon param-file %} `PSP Kinase-Substrate`            | `test_kinase_substrate.tabular`                             |
>    > | {% icon param-file %} `PSP Regulatory`                  | `test_regulatory_sites.tabular`                             |
>    > | {% icon param-file %} `ANOVA alpha-levels`              | `alpha-thresholds`                                          |
>    > | `[PARAM] sample-name extraction pattern`                | `\.\w+\.\d+[A-Z]$`                                          |
>    > | `[PARAM] sample-group extraction pattern`               | `\w+`                                                       |
>    {: .matrix}
>
>    - The two "extraction pattern" PARAM inputs are specified as PERL-compatible Regular Expressions; see [https://rdrr.io/r/base/regex.html](https://rdrr.io/r/base/regex.html) for help.
>
> 4. **Click the `Run Workflow` button.**
>
{: .hands_on}


The output datasets of the preprocessing phase are:
- `phosphoPeptideIntensities` - data matrix of intensities for each phosphopeptide for each sample.
- `enrichGraph` - a PDF graph categorizing the relative frequency at which S, T, and Y residues are phosphorylated.
- `enrichGraph_svg` - an SVG version of `enrichGraph` for incorporation into documents.
- `locProbCutoffGraph` - a PDF graph showing the proportions of inclusion and exclusion of phosphopeptides on the basis of their localization probability scores.
- `locProbCutoffGraph_svg` - an SVG version of `locProbCutoffGraph` for incorporation into documents.
- `filteredData_tabular` - data table (in tabular format) comprising rows of the phosphSites input file that are not flagged as contaminants or reversed sequences.
- `quantData_tabular` - data table (in tabular format) comprising rows of the filteredData file whose localization probability exceeds the Localization Probability Cutoff parameter.
- `mapped_phosphopeptides` - data table (in tabular format) presenting, for each phosphopeptide, the kinase mappings, the mass-spectral intensities for each sample, and the metadata from UniProtKB/SwissProt, phospho-sites, phospho-motifs, and regulatory sites. Data in the columns marked "Domain", "ON\_...", or "...\_PhosphoSite" are available subject to the following terms:
  - "PhosphoSitePlus&reg; (PSP) was created by Cell Signaling Technology Inc. It is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported License (https://creativecommons.org/licenses/by-nc-sa/3.0/). When using PSP data or analyses in printed publications or in online resources, the following acknowledgements must be included: (a) the words 'PhosphoSitePlus(R), www.phosphosite.org' must be included at appropriate places in the text or webpage, and (b) citation of [Hornbeck 2011 (PMID: 25514926)] must be included in the bibliography."
- `preproc_tab` - phosphopeptides annotated with SwissProt and phosphosite metadata, in tabular format. This file is designed to be consumed by the downstream ANOVA tool. Some data in the columns marked "PSP" are available subject to the PhosphoSitePlus terms above.
- `preproc_csv` - phosphopeptides annotated with SwissProt and phosphosite metadata, in CSV format.
- `preproc_sqlite` - SQLite version of `preproc_tab`/`preproc_csv`, augmented with annotations, used by the ANOVA/KSEA tool.

The output datasets of the ANOVA/KSEA phase are:
- `ANOVA_KSEA_report` - a PDF report including and visualizing:
  - Assignments of samples to treatment-classes.
  - An assessment of the similarity of phosphopeptide intensity distributions among samples.
  - An assessment of whether phosphopeptide intensities are distributed unimodally.
  - Distribution of the standard deviations of phosphopeptide intensities.
  - The method and effect of imputing missing values.
  - The effect of quantile normalization of phosphopeptide intensities.
  - ANOVA analysis of phosphopeptide intensities, including a heatmap for the 50 peptides having the lowest adjusted *p*-value less than specified in the `alpha_thresholds` file.
  - Kinase-Substrate Enrichment Analysis results, including *z*-score and FDR for each kinase and, for each kinase, a table, heatmap, and covariance-heatmap of up to 50 substrates.
- `imp_qn_lt_file` - a tabular file presenting imputed, quantile-normalized, log-transformed phosphopeptide intensities, together with some annotation.
- `imputed_data_file` - a tabular file presenting the imputed data without quantile-normalization or log-transformation, together with some annotation.
- `anova_ksea_metadata` - the same annotation plus ANOVA raw and adjusted *p*-values and associated kinases (or motifs) found to be enriched.
- `ksea_sqlite` - SQLite database containing most of the analysis results and annotation for *ad hoc* analysis.

# Preprocessing Notes

The parameters for `MaxQuant Phosphopeptide Preprocessing` are described in the tool help, which you may also see in the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=7b866682d0b1d44c&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6289%2Fmqppep_preproc.xml&changeset_revision=b889e05ce77d&render_repository_actions_for=tool_shed).

Most of the parameters would only require alteration if the defaults are not effective for parsing of the `Phospho (STY)Sites.txt` produced by MaxQuant. However:

- You can use the `Localization probability cutoff` parameter, which, when increased, decreases the number of phosphopeptides quantitated by requiring greater certainty regarding which residues are more likely phosphorylated.
- You can use the `Intensity merge-function` parameter to choose whether to sum or average the intensities of several alternative phosphorylations of one peptide.

# Statistical Notes

The parameters for `MaxQuant Phosphopeptide ANOVA` (which also does KSEA) are described in the tool help, which you may also see in the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=fd3fde90cd863d18&tool_config=%2Fsrv%2Ftoolshed-repos%2Fmain%2F006%2Frepo_6288%2Fmqppep_anova.xml&changeset_revision=514afed1f40d&render_repository_actions_for=tool_shed).

- Considerations regarding the choice for the `Imputation method` parameter: Different methods of imputation are appropriate for different downstream statistical analyses.  Although small random value imputation is the most appropriate method for some univariate data, it perturbs the "center" of multivariate data; applying the median intensity for the treatment for a missing peptide intensity will not (negatively) disturb the sampled between treatments, whereas applying the median across all treatments will not exaggerate the separation of treatments in multivariate space.
- `Sample-name extraction pattern` and `Sample-group extraction pattern` are PERL-compatible regular expressions that presume the sample group to be embedded in the sample name.  If necessary, you will need to name (or rename) your samples in a manner that facilitates writing these expressions.  The default values for these expressions extract as the sample group the alphabetic portion of names like ".C2", where "C" is the sample group (i.e., the treatment) and "2" is the replicate number.
- You can use `Minimum number of values per sample-group` to remove from the analysis peptides that are poorly represented in some sample groups.
- You can use `Filter sample-groups` to remove from your analysis sample-groups that you do not wish to be considered, e.g., quality-control samples.
  - If you wish to remove samples, the `Sample-group matching mode` specifies whether you wish to ignore case and which type of matching you prefer: exact match, "extended grep" match, or "PERL-compatible" match.
  - You also will specify `Sample-group matching pattern`, a comma-separated list of patterns of the indicated matching type.
- You may specify `Minimum number of kinase-substrates for KSEA` to exclude kinases represented by very few substrates.
- You may use `KSEA threshold level` to specify the minimum FDR at which a kinase will be considered to be enriched; *the default choice of 0.05 is arbitrary and may exclude kinases that are interesting.* The KSEA FDR perhaps should not be treated as conservatively as would be appropriate for hypothesis testing. For example, at an FDR of 0.05, for every 20 kinases that on discards, 19 are likely truly enriched.
- You can set `Use abs(log2(fold-change)) for KSEA` to TRUE if you wish to consider the magnitude but not direction of changes in phosphopeptide intensity (e.g., if you have reason to believe that some interesting phosphopeptides may be phosphorylated at the expense of other substrates of a given kinase.
- You can set `Minimum quality of substrates for KSEA` to higher values to exclude from KSEA analysis phosphopeptides having a relatively high combination of *p*-value and relatively more missing values:
  - $$
    \textit{quality}_j
      = \frac{1 + o_{j}}{v_{j}(1 + o_{j}) + 0.005}
  $$<br />
  where:
  - $$o_j$$ is the minimum number of non-missing observations per sample group for substrate $$j$$ for all sample groups, and
  - $$v_j$$ is the FDR-adjusted ANOVA $$p$$-value for substrate $$j$$.

# Authorship and repository

To build these tools, Arthur C. Eschenlauer ([ORCiD 0000-0002-2882-0508](https://orcid.org/0000-0002-2882-0508)) adapted scripts written by Nicholas A. Graham ([ORCiD 0000-0002-6811-1941](https://orcid.org/0000-0002-6811-1941)) and Larry C. Cheng ([0000-0002-6922-6433](https://orcid.org/0000-0002-6922-6433)).

The source code for these tools is at [https://github.com/galaxyproteomics/tools-galaxyp/tree/master/tools/mqppep](https://github.com/galaxyproteomics/tools-galaxyp/tree/master/tools/mqppep).

# Conclusion

Sum up the tutorial and the key takeaways here. We encourage adding an overview image of the
pipeline used.
