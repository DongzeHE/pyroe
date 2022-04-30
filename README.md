# pyroe

## Introduction

[`Alevin-fry`](https://github.com/COMBINE-lab/alevin-fry) is a fast, accurate, and memory frugal quantification tool for preprocessing single-cell RNA-sequencing data. Detailed information can be found in the alevin-fry [pre-print](https://www.biorxiv.org/content/10.1101/2021.06.29.450377v2), and [paper](https://www.nature.com/articles/s41592-022-01408-3).

The `pyroe` package provides useful functions for analyzing single-cell or single-nucleus RNA-sequencing data using `alevin-fry`, which consists of

1. preparing the *splici* reference for the `USA` mode of `alevin-fry`, which will export a unspliced, a spliced, and an ambiguous molecule count for each gene within each cell.
2. fetching and loading the preprocessed quantification results of `alevin-fry` into python as an [`AnnData`](https://anndata.readthedocs.io/en/latest/) object.

## Installation
The `pyroe` package can be accessed from its [github repository](https://github.com/COMBINE-lab/pyroe), installed via [`pip`](https://pip.pypa.io/en/stable/). To install the `pyroe` package via `pip` use the command:

```
pip install pyroe
```

To make use of the `load_fry` function (which, itself, installs [scanpy](https://scanpy.readthedocs.io/en/stable/)), you should also be sure to install the package with the `scanpy` extra:

```
pip install pyroe[scanpy]
```

Alternatively, `pyroe` can be installed via `bioconda`, which will automatically install the variant of the package including `load_fry`, and will
also install `bedtools` to enable faster construction of the *splici* reference (see below).  This installation can be performed with the command:

```
conda install pyroe
```

with the appropriate bioconda channel in the conda channel list.


## Preparing a splici index for quantification with alevin-fry

The USA mode in alevin-fry requires a special index reference, which is called the *splici* reference. The *splici* reference contains the spliced transcripts plus the intronic sequences of each gene. The `make_splici_txome()` function is designed to make the *splici* reference by taking a genome FASTA file and a gene annotation GTF file as the input. Details about the *splici* can be found in Section S2 of the supplementary file of the [alevin-fry paper](https://www.nature.com/articles/s41592-022-01408-3). To run pyroe, you also need to specify the read length argument `read_length` of the experiment you are working on and the flank trimming length `flank_trim_length`. A final flank length will be computed as the difference between the read_length and flank trimming length and will be attached to the ends of each intron to absorb the intron-exon junctional reads.

Following is an example of calling the `pyroe` to make the *splici* index reference. The final flank length is calculated as the difference between the read length and the flank_trim_length, i.e., 5-2=3. This function allows you to add extra spliced and unspliced sequences to the *splici* index, which will be useful when some unannotated sequences, such as mitochondrial genes, are important for your experiment. **Note** : to make `pyroe` work more quickly, it is recommended to have the latest version of [`bedtools`](https://bedtools.readthedocs.io/en/latest/) ([Aaron R. Quinlan and Ira M. Hall, 2010](https://doi.org/10.1093/bioinformatics/btq033)) installed.

```
pyroe make-splici extdata/small_example_genome.fa extdata/small_example.gtf 5 splici_txome \
      --flank-trim-length 2 --filename-prefix transcriptome_splici --dedup-seqs
```

The `pyroe` program writes two files to your specified output directory `output_dir`. They are 
- A FASTA file that stores the extracted splici sequences.
- A three columns' transcript-name-to-gene-name file that stores the name of each transcript in the splici index reference, their corresponding gene name, and the splicing status (`S` for spliced and `U` for unspliced) of those transcripts.

### Full usage

```
usage: pyroe make-splici [-h] [--filename-prefix FILENAME_PREFIX]
                         [--flank-trim-length FLANK_TRIM_LENGTH]
                         [--extra-spliced EXTRA_SPLICED]
                         [--extra-unspliced EXTRA_UNSPLICED]
                         [--bt-path BT_PATH] [--dedup-seqs] [--no-bt]
                         [--no-flanking-merge]
                         genome-path gtf-path read-length output-dir

positional arguments:
  genome-path           The path to a gtf file.
  gtf-path              The path to a gtf file.
  read-length           The read length of the single-cell experiment 
                          being processed (determines flank size).
  output-dir            The output directory where splici reference 
                          files will be written.

optional arguments:
  -h, --help            show this help message and exit
  --filename-prefix FILENAME_PREFIX
                        The file name prefix of the generated output files.
  --flank-trim-length FLANK_TRIM_LENGTH
                        Determines the amount subtracted from the read length
                        to get the flank length.
  --extra-spliced EXTRA_SPLICED
                        The path to an extra spliced sequence fasta file.
  --extra-unspliced EXTRA_UNSPLICED
                        The path to an extra unspliced sequence fasta file.
  --bt-path BT_PATH     The path to bedtools v2.30.0 or greater.
  --dedup-seqs          A flag indicates whether identical sequences will be
                          deduplicated.
  --no-bt               A flag indicates whether to disable bedtools.
  --no-flanking-merge   A flag indicates whether introns will be merged after
                          adding flanking length.
```

### the *splici* index

The *splici* index of a given species consists of the transcriptome of the species, i.e., the spliced transcripts, and the intronic sequences of the species. Within a gene, if the flanked intronic sequences overlap with each other, the overlapped intronic sequences will be collapsed as a single intronic sequence to make sure each base will appear only once in the intronic sequences. For more detailed information, please check the section S2 in the supplementary file of [alevin-fry manuscript](https://www.biorxiv.org/content/10.1101/2021.06.29.450377v2).

## Processing alevin-fry quantification result

The quantification result of alevin-fry can be loaded into python by the `load_fry()` function. This function takes a output directory returned by `alevin-fry quant` command as the minimum input, and load the quantification result as an `AnnData` object. When processing USA mode result, it assumes that the data comes from a single-cell RNA-sequencing experiment. If one wants to process single-nucleus RNA-sequencing data or prepare the single-cell data for RNA-velocity analysis, the `output_format` argument should be set as `snRNA` or `velocity` correspondingly. One can also define customized output format, see the Full Usage section for detail.

### Full Usage

load alevin-fry quantification result into an AnnData object

#### Required Parameters

frydir : `str`
    The path to a output directory returned by alevin-fry quant command. \\
    The directory containing the alevin-fry quantification (i.e. the the quant.json file & alevin subdirectory).

#### Optional Parameters

output_format : `str` or `dict`
    A string represents one of the pre-defined output formats, which are "scRNA", "snRNA" and "velocity". \\
    If a customized format of the returned `AnnData` is needed, one can pass a Dictionary.\\
    See Notes section for details.

quiet : `bool` (default: `True`)
    True if function should be quiet.
    False if messages (including error messages) should be printed out. 
    
nonzero : `bool` (default: `False`)
    True if cells with non-zero expression value across all genes should be filtered in each layer.
    False if unexpressed genes should be kept.

#### Notes

The `output_format` argument takes either a dictionary that defines the customized format or 
a string that represents one of the pre-defined format of the returned `AnnData` object.

Each of the pre-defined formats contains a `X` field and some optional extra `AnnData.layers` 
obtained from the submatrices representing unspliced (U), spliced (S) and ambiguous (A) counts 
returned by alevin-fry. 

The following formats are defined:

* "scRNA": \
    This format is recommended for single cell RNA-sequencing experiments. 
    It returns a `X` field that contains the S+A count of each gene in each cell without any extra layers.

* "snRNA": \
    This format is recommended for single nucleus RNA-sequencing experiments. 
    It returns a `X` field that contains the U+S+A count of each gene in each cell without any extra layers.

* "raw": \
    This format uses the S count matrix as the `X` field and put the U, S, and A counts into three 
    separate layers, which are "unspliced", "spliced" and "ambiguous".

* "velocity": \
    This format is the same as "scRNA", except it contains two extra layers: the "spliced" layer, 
    which contains the S+A counts, and the "unspliced" layer, which contains the U counts.

A custom output format can be defined using a Dictionary specifying the desired format of the output `Anndata` object.  
If the input is not a USA mode quantification directory, this parameter is ignored
and the count matrix is returned in the `X` field of the returned `AnnData` object.  If the input
quantification directory contains a USA mode quantification, then there are 3 sub-matrices that can 
be referenced in the dictionary; 'U', 'S', 'A' containing, respectively, unspliced, spliced and 
ambiguous counts.  The dictionary should have entries of the form `key` (str) : `value` (list[str]).
The following constraints apply : there should be one key-value pair with the key `X`, the resulting
counts will be returned in the `X` field of the AnnData object. There can be an arbitrary number
of other key-value pairs, but each will be returned as a layer of the resulting AnnData object.
Within the key-value pairs, the key refers to the layer name that will be given to the combined 
count matrix upon output, and the value should be a subset of `['U', 'S', 'A']` that defines 
which sub-matrices should be summed.  For example:
`{'X' : ['S', 'A'], 'unspliced' : ['U']}`
will result in a return AnnData object where the X field has a matrix in which each entry 
corresponds to the summed spliced and ambiguous counts for each gene in each cell, and there
is an additional "unspliced" layer, whose counts are taken directly from the unspliced sub-matrix.

#### Returns
An AnnData object with X and layers corresponding to the requested `output_format`.

## Fetching and loading preprocessed quantification results
The raw data for many single-cell and single-nucleus RNA-seq experiments is publicly available.  However, certain datasets are used _again and again_, to demonstrate data processing in tutorials, as benchmark datasets for novel methods (e.g. for clustering, dimensionality reduction, cell type identification, etc.).  In particular, 10x Genomics hosts various publicly available datasets generated using their technology and processed via their Cell Ranger software [on their website for download](https://www.10xgenomics.com/resources/datasets).

We have created a [Nextflow](https://www.nextflow.io)-based `alevin-fry` workflow that one can use to easily quantify single-cell RNA-sequencing data in a single workflow.  The pipeline can be found [here](https://github.com/COMBINE-lab/10x-requant).  To test out this initial pipeline, we have begun to reprocess the publicly-available datasets collected from the 10x website. We have focused the initial effort on standard single-cell and single-nucleus gene-expression data generated using the Chromium v2 and v3 chemistries, but hope to expand the pipeline to more complex protocols soon (e.g. feature barcoding experiments) and process those data as well.  We note that these more complex protocols can already be processed with `alevin-fry` (see the [alevin-fry tutorials](https://combine-lab.github.io/alevin-fry-tutorials/)), but these have just not yet been incorprated into the automated Nextflow-based workflow linked above.

We provide two python functions, `fetch_processed_quant()`, which can fetch the processed quantification results and store them to a local folder, and `load_processed_quant()`, which can fetch the quantification results and load them into python as `AnnData` objects. We also provide a CLI for fetching quantification results.

### Full usage

```
usage: pyroe fetch-quant [-h] [--fetch_dir FETCH_DIR] [--force] [--keep_tar]
                         [--quiet]
                         dataset-ids [dataset-ids ...]

positional arguments:
  dataset-ids           The ids of the datasets to fetch

optional arguments:
  -h, --help            show this help message and exit
  --fetch_dir FETCH_DIR
                        The path to a directory for storing fetched datasets.
  --force               A flag indicates whether existing datasets will be redownloaded by force.
  --keep_tar          A flag indicates whether fetched tar files will be deleted.
  --quiet               A flag indicates whether help messaged should not be printed.

1. 500 Human PBMCs, 3' LT v3.1, Chromium Controller
2. 500 Human PBMCs, 3' LT v3.1, Chromium X
3. 1k PBMCs from a Healthy Donor (v3 chemistry)
4. 10k PBMCs from a Healthy Donor (v3 chemistry)
5. 10k Human PBMCs, 3' v3.1, Chromium X
6. 10k Human PBMCs, 3' v3.1, Chromium Controller
7. 10k Peripheral blood mononuclear cells (PBMCs) from a healthy donor, Single Indexed
8. 10k Peripheral blood mononuclear cells (PBMCs) from a healthy donor, Dual Indexed
9. 20k Human PBMCs, 3' HT v3.1, Chromium X
10. PBMCs from EDTA-Treated Blood Collection Tubes Isolated via SepMate-Ficoll Gradient (3' v3.1 Chemistry)
11. PBMCs from Heparin-Treated Blood Collection Tubes Isolated via SepMate-Ficoll Gradient (3' v3.1 Chemistry)
12. PBMCs from ACD-A Treated Blood Collection Tubes Isolated via SepMate-Ficoll Gradient (3' v3.1 Chemistry)
13. PBMCs from Citrate-Treated Blood Collection Tubes Isolated via SepMate-Ficoll Gradient (3' v3.1 Chemistry)
14. PBMCs from Citrate-Treated Cell Preparation Tubes (3' v3.1 Chemistry)
15. PBMCs from a Healthy Donor: Whole Transcriptome Analysis
16. Whole Blood RBC Lysis for PBMCs and Neutrophils, Granulocytes, 3'
17. Peripheral blood mononuclear cells (PBMCs) from a healthy donor - Manual (channel 5)
18. Peripheral blood mononuclear cells (PBMCs) from a healthy donor - Manual (channel 1)
19. Peripheral blood mononuclear cells (PBMCs) from a healthy donor - Chromium Connect (channel 5)
20. Peripheral blood mononuclear cells (PBMCs) from a healthy donor - Chromium Connect (channel 1)
21. Hodgkin's Lymphoma, Dissociated Tumor: Whole Transcriptome Analysis
22. 200 Sorted Cells from Human Glioblastoma Multiforme, 3’ LT v3.1
23. 750 Sorted Cells from Human Invasive Ductal Carcinoma, 3’ LT v3.1
24. 2k Sorted Cells from Human Glioblastoma Multiforme, 3’ v3.1
25. 7.5k Sorted Cells from Human Invasive Ductal Carcinoma, 3’ v3.1
26. Human Glioblastoma Multiforme: 3’v3 Whole Transcriptome Analysis
27. 1k Brain Cells from an E18 Mouse (v3 chemistry)
28. 10k Brain Cells from an E18 Mouse (v3 chemistry)
29. 1k Heart Cells from an E18 mouse (v3 chemistry)
30. 10k Heart Cells from an E18 mouse (v3 chemistry)
31. 10k Mouse E18 Combined Cortex, Hippocampus and Subventricular Zone Cells, Single Indexed
32. 10k Mouse E18 Combined Cortex, Hippocampus and Subventricular Zone Cells, Dual Indexed
33. 1k PBMCs from a Healthy Donor (v2 chemistry)
34. 1k Brain Cells from an E18 Mouse (v2 chemistry)
35. 1k Heart Cells from an E18 mouse (v2 chemistry)
```
