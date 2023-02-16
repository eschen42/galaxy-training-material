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
time_estimation: 1H
key_points:
- The tools automate the analysis of differential phosphorylation and kinase-substrate enrichment analysis.
- A rudimentary knowledge of regular expressions is required to set some parameters.
tags: [DIA, human]
subtopic: id-quant
contributors:
- eschen42

---

# Introduction

Phosphopeptides constitute a minority of the peptides prepared by proteolysis for shotgun MS analysis;
enrichment for phosphopeptides simplifies differential phosphorylation analysis ({% cite Cheng_2018 %}).

MaxQuant ({% cite Cox_2008 %}; {% cite Cox_2014 %}) can be used effectively to perform label-free
identification and quantitation phopsphopeptides.  However, a data-processing software pipeline is necessary to analyze and interpret these results.

This tutorial introduces:

1. Obtaining the prerequisite static files for this pipeline.
1. Running MaxQuant within an subworkflow to create a customized `mqpar.xml` file.
1. Running MaxQuant with a customized `mqpar.xml` file to create the `Phospho (STY)Sites.txt` file.
1. Applying preprocessing to the phosphopeptide sites identified and quantitatie by MaxQuant.
1. Applying statistical analysis to the preprocessed data.

This tutorial **focuses on human phosphoproteomics**. However, it might not be prohibitively difficult to use different static files to investigate phosphopeptides other organisms.

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

The data processing of Thermo-Fisher RAW LC-MS data files to ANOVA and KSEA results is summarized ih the following figure:

<figure id="figure-1">
  <img
    width="700"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphenrichment/overview_flowchart.svg"
    alt="The MaxQuant Phosphoenrichment Data Processing Pipeline comprises these steps: construction of `mqpar.xml`; running of MaxQuant; extraction of the `Phospho (STY)Sites.txt` file; preprocessing; statistical analysis with ANOVA and KSEA."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 1:</span> Overview of the MaxQuant Phosphoenrichment Data Processing Pipeline
  </figcaption>
</figure>

This extensive process will be presented here as a sequence of several phases:

1. Gathering (or making) the (several) static input files required.
2. Uploading the Thermo RAW LC-MS data files and a tabular file describing the samples.
3. Running MaxQuant, in two steps:
  - Synthesizing the MaxQuant `mqpar.xml` parameters file using the [**MaxQuant**](https://toolshed.g2.bx.psu.edu/view/galaxyp/maxquant/4d17fcdeb8ff) tool (e.g., see [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=0c6018b0e3966b99&changeset_revision=837224ad1694&tool_config=/srv/toolshed-repos/main/000/repo_410/maxquant.xml)).
  - Running MaxQuant from `mqpar.xml` using the [**MaxQuant (using mqpar.xml)**](https://toolshed.g2.bx.psu.edu/view/galaxyp/maxquant_mqpar/9cb7dcc07dae) tool {e.g., see [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=0c6018b0e3966b99&changeset_revision=837224ad1694&tool_config=/srv/toolshed-repos/main/000/repo_410/maxquant_mqpar.xml)).
4. Applying preprocessing (phosphosite-localization and protein identification) to the sites identified by MaxQuant using the [**MaxQuant Phosphopeptide Preprocessing**](https://toolshed.g2.bx.psu.edu/view/galaxyp/mqppep_preproc/5c2beb4eb41c) tool (e.g., see [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=7b866682d0b1d44c&changeset_revision=5c2beb4eb41c&tool_config=/srv/toolshed-repos/main/006/repo_6289/mqppep_preproc.xml)).
5. Performing normalization, imputation, ANOVA and KSEA (Kinase-Substrate Enrichment Analysis) using the [**MaxQuant Phosphopeptide ANOVA**](https://toolshed.g2.bx.psu.edu/view/galaxyp/mqppep_anova/2d9f216c1048) tool (e.g., see [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=fd3fde90cd863d18&changeset_revision=2d9f216c1048&tool_config=/srv/toolshed-repos/main/006/repo_6288/mqppep_anova.xml)).

## Four Related Workflows

Four closely related workflows implementing phases 3-5 of phosphoproteomic enrichment data processing workflow are presented in this tutorial, in varying degrees of detail:

[**Download the "Quant" workflow**](tutorials/workflow-Phosphproteomic_Enrichment_Pipeline_-_Quant.ga); this performs the initial stages (*phase 3*) of the workflow: Prepare mqpar.xml and run MaxQuant.

<figure id="figure-2">
  <img
    height="250"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphenrichment/overview_flowchart_phase3.svg"
    alt="The Quant Phase of the MaxQuant Phosphoenrichment Data Processing Pipeline comprises these steps: construction of `mqpar.xml`; running of MaxQuant; extraction of the `Phospho (STY)Sites.txt` file."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 2:</span> The "Quant" phase of the pipeline
  </figcaption>
</figure>

[**Download the "Synthesize mqpar.xml" subworkflow**](subworkflow-synthesize_mqpar.xml_for_MaxQuant.ga); this subworkflow is invoked by the "Quant" workflow, produces a list of mzXML and fully-specified `mqpar.xml` for MaxQuant run.  A copy of this subworkflow is included and annotated here for inspection, but it will not be discussed further in this tutorial.

<figure id="figure-3">
  <img
    height="250"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphenrichment/overview_flowchart_phase3a.svg"
    alt="The mqpar.xml contruction subworkflow from the MaxQuant Phosphoenrichment Data Processing Pipeline."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 3:</span> The `mqpar.xml` construction subworkflow of "Quant" phase
  </figcaption>
</figure>

[**Download the "Postprocessing" workflow**](workflow-Phosphproteomic_Enrichment_Pipeline_-_Postprocessing.ga); this performs the post-MaxQuant half (*phases 4 and 5*) of the workflow: Preprocess results and produce ANOVA/KSEA report.

<figure id="figure-4">
  <img
    height="250"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphenrichment/overview_flowchart_phases4and5.svg"
    alt="The Postprocessing Phases of the MaxQuant Phosphoenrichment Data Processing Pipeline comprises these steps: preprocessing; statistical analysis with ANOVA and KSEA."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 4:</span> `mqpar.xml` Postprocessing phases of the pipeline
  </figcaption>
</figure>

[**Download the "Overview" workflow**](workflow-Phosphproteomic_Enrichment_Pipeline_-_Overview.ga); this workflow includes of all the steps in the "Quant" and "Postprocessing" workflows.  This tutorial discusses the "Quant" and "Postprocessing" workflows instead of discussing the "Overview", but it may be convenient to run the "Overview" workflow once you are familiar with how the other two behave.

Figure 1 at [the beginning of this Overview section](#overview) depicts this workflow.

## Static inputs

Several files will change little if at all from one run of the pipeline to the next.  You will probably get them (directly or from a colleague) and put them into a Galaxy history from which you can copy them when starting a new run of the pipeline.

1. FASTA files
  - One or more FASTA files bearing the sequences of peptides that you expect to find; you may want files that cover all isoforms.
  - One or more FASTA files bearing the sequences of contaminants that you expect may be present.
1. An `mqpar.xml` file generated by MaxQuant with most of the general settings that you require; e.g., you can save this from the MaxQuant desktop program.
1. A file of alpha levels to be used as thresholds in ANOVA analysis.
1. The four phosphosite databases:
  - PSP Kinase-Substrate sites
  - PSP Regulatory sites
  - HPRD pSTY motifs
  - NetworKIN sites

## Run-specific inputs

For each run of MaxQuant, you will need to upload the Thermo RAW LC-MS data files.

Also, you will need the "Name-to-Fraction-Experiment Map" comprising described below.  This file is a simplification of the "Experimental Design" file

## Synthesizing `mqpar.xml`

Many MaxQuant parameters that do not change from one run to the next.
Save the invariant parameters in your MaxQuant `mqpar.xml` parameters file that you use as input for this step; see ["Static input files"](#static-input-files) above.

The synthesis step creates a MaxQuant `mqpar.xml` parameters file customized for a specific run.
An individual run of the MaxQuant tool also needs specification of the FASTA files, data files, and an experimental design file for that run.
The first part of the MaxQuant workflow presented here "splices" the invariant parts of `mqpar.xml` with these paths.

## Running MaxQuant

The actual running of MaxQuant with parameters provided in the `mqpar.xml` takes a very long time, which is why it is better to start with an `mqpar.xml` file in which you have confidence so that runs will be reliable and reproducible.  Among the outputs of MaxQuant will be the `Phospho (STY)Sites.txt` file that will be used by the next step.

## Preprocessing MQ results

The "preprocessing" step:

- Searches the FASTA file(s) of expected protein sequences for matches to the phosphopeptides detected, characterized, and quantitated by MaxQuant.
- Computes "localization probability" calculations to indicate the most probable location(s) for phosphorylaton of the detected peptides.
- Produces outputs tailored to the statistical analysis step

## ANOVA and KSEA

The final step covered here is performing a pair of statistical analyses after some preparatory steps:

- Perform quantile normalization ({% cite Bolstad_2003 %}).
- Perform imputation of missing values by one of several possible methods.
- Perform additional annotation with phosphorylation-site databases.
- **ANOVA** (analysis of variance) is a **univariate** technique used to identify **each phosphopeptide for which there is considerable evidence against the null hypothesis** that their variation among the several treatements occurs by random chance.  There are several challenges to interpreting these data:
  - The data reflect the abundance of particular protein modifications, which may result in relative differences in activity, but they do not give any sense regarding the regulatory signals that led to these differences.
  - Inter-sample, intra-treatement variation for individual phosphopeptides can be extensive (indeed, missing values are common).  **A small treatment effect signal can be swamped by the "noise".**
- **KSEA** (Kinase-Substrate Enrichment Analysis) groups by kinase the subtle differences among the individual phosphopeptide substrates and **estimates the likelihood that the substrates of each kinase differ more than would be expected at random**.
  - In effect, even though it may be likely that individual substrates' differences might be expected because of random variation, the likelihood of the aggregation of substrates of a given kinase differing by random chance as much as is observed may be rather unlikely.
- Both of these analysies are performed by the `MaxQuant Phosphopeptide ANOVA` tool 

<figure id="figure-5">
  <img
    width="700"
    src="{{ site.baseurl }}/topics/proteomics/images/maxquant-phosphenrichment/KSEA_impl_flowchart.svg"
    alt="Flowchart showing steps in statistical analysis with ANOVA and KSEA."
    loading="lazy">
  <figcaption>
    <span class="figcaption-prefix">Figure 5:</span> ANOVA and KSEA processing steps
  </figcaption>
</figure>

# 1. Static inputs

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
>   - Once you have agreed to the licenses and followed the proceure, the scripts can run.
> - The scripts run under Linux, where all of the dependencies may be met easily through Conda.
>   - Some additional work will be necessary to run them under Windows or MacOS; in such cases, it may be easiest to run them in a Docker container.
>
{: .details}

## FASTA files for expected experimental and contaminant sequences

You will want to have:

  - one or more files of expected peptide sequences for the MaxQuant step and for subsequent peptide/protein searches during the preprocessing step.
    - You may want an "all isozymes" version because your experimental data likely include phosphopeptides from all isozyemes, not just the "most representative isozyme".
  - zero or more files of expected contaminating peptide sequences for the MaxQuant step.

> <details-title>Retrieving FASTA for all human isoforms from UniProt</details-title>
>
> Follow the process described at
> [https://www.uniprot.org/help/retrieve\_sets](https://www.uniprot.org/help/retrieve_sets)
> to run some scripts to download all isoforms of a proteome.
>
> For example, to download all isoforms of the human protome:
>
> - Using a web browser:
>   - Navigate to [https://www.uniprot.org/uniprotkb?query=proteome:UP000005640](https://www.uniprot.org/uniprotkb?query=proteome:UP000005640)
>   - Click the `Download` button (which looks like this &nbsp; <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32" fill="blue" width="14" height="14"><path d="M27.6 21h3.9v7.7H.5v-7.8h3.9v4h23.2v-4zm-3.9-9.7L16 22.9 8.2 11.3h5.9V1.6h3.8v9.7h5.9z"></path></svg>)
>   - Choose:
>     - `Download all`
>     - Format `FASTA (cononical & isoform)`
>     - Compressed: `yes`
>   - Click download, and a "gzipped" FASTA file will download.
>   - From the command line, uncompress the download with the `gzip -d` command.
>     - Alternatively, you may use a multifunctional decompression program like 7-zip ([https://www.7-zip.org/](https://www.7-zip.org/)).
> - Alternatively, if you know the proteome ID and you are familiar with the command line and `wget`, you can use a command like this:
>   ```
wget "http://rest.uniprot.org/uniprotkb/stream?format=fasta&includeIsoform=true&query=%28proteome%3AUP000005640%29"
```
>
{: .details}

## MaxQuant `mqpar.xml` parameters file

The `mqpar.xml` file has myriad parameters for the many steps in the sophisticated pipeline that the MaxQuant tool implements.  A great place to start familiarizing yourself with these parameters is in [the **Label-free data analysis using MaxQuant** tutorial]({{ site.baseurl }}/topics/proteomics/tutorials/maxquant-label-free/tutorial.html#maxquant-analysis).

Although this file does include paths to your FASTA files, your LC-MS data files, and your experimental design details, it also contains many parameters that you will never change from experiment to experiment.  This is why you want to set up a reliable starter `mqpar.xml` file; assignment of the FASTA files, data files, and experimental design are part of the first MaxQuant step, so don't worry about them (or bother putting them into the MaxQuatnt GUI) at this time.

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
>    For further discuassion, please see
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
>   - For this tutorial, however, **we will use the "common Reposotory of Adventitious Propteins"**; obtain this as described in the
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
>   - For tissue culture, you may wish to add proteomes of *likely* *Mycoplasma* contaminants, by either of the methods described above, decorating with the CONTAMINANT flag as described in section .
>     For an explanation of the considerations to make, see the "Custom Contaminant Detection (Mycoplasma)" section of({% cite Bielow_2016 %}).
>   - Note that the decisions made here will influence the PTXQC report produced by the step in which MaxQuant runs.
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
> > - `common Repository of Adentitious Proteins.fasta`
> >
> {: .solution}
{: .question}

# 2. Run-specific inputs

## Create history with data files

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
{: .hands_on}

> <hands-on-title> Create a dataset collection with the LC-MS data </hands-on-title>
>
> - Build **a "dataset collection" containing the Thermo RAW files**, naming it `pST Thermo RAW`.  (Be *sure* only to include the Thermo RAW files; otherwise, the collection will not work properly.)
>
> {% snippet faqs/galaxy/collections_build_list.md %}
{: .hands_on}


## Add "Name-to-Fraction-Experiment Map"

To create "experimental design" input for MaxQuant, the workflow uses a tabular file mapping sample name to fraction and experiment.  It must have three named columns: `Name`, `Fraction`, and `Experiment`.

1. The first column, `Name`, should have names *exactly* matching the Thermo RAW file names, minus any .raw or .thermo.raw file extensions.
2. The second column, `Fraction`, distinguishes fractions among the same sample.
3. The third column (which will be the same for all fractions of one sample), `Experiment`, affects the labeling of output columns.
  - It may be helpful to encode the treatment and replica into the value in this column.

> <hands-on-title> Create a name-to-fraction-experiment map dataset </hands-on-title>
> Upload (by "Paste/Fetch") **a tabular file mapping sample name to fraction and experiment**, named `name2fracexpt_for_pST.tabular`.
>
> > | Name      | Fraction | Experiment       |
> > | --------------------------------------- |
> > | QE03659-2 | 1        | 22Rv1.AR.3B      |
> > | QE03659   | 1        | 22Rv1.AR.3A      |
> > | QE03670-2 | 1        | PARCB-5.AVPC.14B |
> > | QE03670   | 1        | PARCB-5.AVPC.14A |
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

## Copy the static inputs

Copy the static inputs (from section ["save your static files"](#save-your-static-files) above) into the history where you uploaded the data files and the map.

# 3. MaxQuant workflow

## Option 1: "Overview" complete workflow

One option you have is to run the entire workflow (MaxQuant, preprocessing, and statistical analysis) in a single workflow run, the "Overview" workflow.  The "Overview" workflow comprises the steps from the "Quant" and "Postprocessing" partial workflows.  Eventually, you may become familiar enough with the process to adapt and use this workflow, but for the sake of explaining the process, we will cover the second option in the rest of this tutorial.

## Option 2: "Quant" partial workflow

A second option is to run the "Quant" partial workflow and postpone running the "Postprocessing" partial workflow.  To simplify the presentation, this option is the one described in this tutorial.

> <hands-on-title>Quant (phase 3)</hands-on-title>
>
> 1. **Import the workflow** into Galaxy
>    - Copy (e.g. via right-click) [the URL of the "Quant" workflow]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphproteomic_Enrichment_Pipeline_-_Quant.ga) and download it to your computer.
>    - Import the workflow into Galaxy
>
>    {% snippet faqs/galaxy/workflows_import.md %}
>
> 2. **Set up the workflow invocation**
>    > {% snippet faqs/galaxy/workflows_run.md %}
>
>    Enter the following parameters:
>
>    > | Input                                                   | Value                                            |
>    > | ------------------------------------------------------- | ------------------------------------------------ |
>    > | `Send results to a new history`                         | `Yes`                                            |
>    > | `History name`                                          | `mqppep tutorial run`                            |
>    > | {% icon param-file %} `FASTA valid phosphopeptides`     | `HumanProteome_UP000005640_AllIsozymes.fasta`    |
>    > | {% icon param-file %} `FASTA contaminants`              | `common Repository of Adentitious Proteins.fasta`|
>    > | `[PARAM] FASTA id parse rule`                           | `>([^\s]*)`                                      |
>    > | `[PARAM] FASTA desc parse rule`                         | `>(*)`                                           |
>    > | {% icon param-file %} `Name-to-Fraction-Experiment Map` | `name2fracexpt_for_pST.tabular`                  |
>    > | {% icon param-file %} `Thermo RAW collection`           | `pST Thermo RAW`                                 |
>    > | {% icon param-file %} `General mqpar.xml`               | `mqpar.xml`                                      |
>    {: .matrix}
>
> 3. **Click the `Run Workflow` button.**
>
> 4. Wait a *long* time.  (Perhaps surprisingly) you can monitor progress of MaxQuant by looking at the contents of the (running) dataset:
>    - {% icon param-file %} `VIEW PROGRESS HERE`.
>
> 5. The output datasets of immediate interest are:
>    - {% icon param-file %} `phospho_STY_sites.txt`, which  you will take to the Postprocessing workflow.
>    - {% icon param-file %} `PTXQC_report`; to determine how quality considerations from this report should influence your subsequent analysis, study {% cite Bielow_2016 %} carefully.
>
{: .hands_on}

# 4. Postprocessing workflow

## Option 2 (continued): "Postprocessing" workflow

After you have run the Quant workflow, you will have the {% icon param-file %} `phospho_STY_sites.txt` dataset in hand.  Invocation of the "Postprocessing" workflow initiates a pipeline invoking two substeps:

- Filtering of phosphosite localization probabilities, imputation of missing values, and normalization.
- Statistical analysis with ANOVA and KSEA.

After the pipeline has run, each of these substeps may be re-run with changed parameters, if desired.

> <hands-on-title>Quant (phases 4 and 5)</hands-on-title>
>
> 1. **Import the workflow** into Galaxy
>    - Copy (e.g. via right-click) [the URL of the "Postprocessing" workflow]({{ site.baseurl }}{{ page.dir }}workflows/workflow-Phosphproteomic_Enrichment_Pipeline_-_Postprocessing.ga) and download it to your computer.
>    - Import the workflow into Galaxy
>
> 2. **Set up the workflow invocation**
>    - If you wish to change any of the tool parameters, you can click `Expand to full workflow form.` at the bottom of the form and then expand the two tool steps to reveal the parameters.
>    - The parameters for `MaxQuant Preprocessing` are described in the tool help, which you may also see in the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=7b866682d0b1d44c&changeset_revision=5c2beb4eb41c&tool_config=/srv/toolshed-repos/main/006/repo_6289/mqppep_preproc.xml).
>    - The parameters for `MaxQuant ANOVA` (which also does KSEA) are described in the tool help, which you may also see in the [**tool description**](https://toolshed.g2.bx.psu.edu/repository/display_tool?repository_id=fd3fde90cd863d18&changeset_revision=2d9f216c1048&tool_config=/srv/toolshed-repos/main/006/repo_6288/mqppep_anova.xml).
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
> 4. **Click the `Run Workflow` button.**
>
{: .hands_on}

# Conclusion

Sum up the tutorial and the key takeaways here. We encourage adding an overview image of the
pipeline used.
