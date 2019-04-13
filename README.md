# bcbio-pre-cancer-config

Configuration to use [bcbio workflow](https://bcbio-nextgen.readthedocs.io/en/latest/contents/pipelines.html#cancer-variant-calling) for analysis of pre-cancer FFPE samples without matched normal for somatic, germline and copy number variant calling.

Method assumes WGS or targeted hybridization library sequencing preparation such as Agilent XTHS.

This workflow leverages unique molecular identifiers (UMIs) for additional error correction and PCR duplicate removal. It also uses the population database approach of filtering germline samples described in the [bcbio documentation](https://bcbio-nextgen.readthedocs.io/en/latest/contents/pipelines.html#cancer-variant-calling) as well as in this [benchmarking blog post](http://bcb.io/2015/03/05/cancerval/).

## Configuration

The primary purpose of this repository is to document a use case of bcbio for tumor-only cancer variant calling using UMIs. 

The configuration example is in the form of a yaml file and in this repository is named: `bcbio_pre-cancer.yaml`

### Sample information

At the minimum you will need to edit information about your samples:
```
- description: sample1
  files: [/path/sample1_R1.fq.gz, /path/to/sample1_R2.fq.gz]
  metadata:
    batch: [sample1]
```

### Regions for analysis 

To indicate a region to calculate coverage in QC metrics, indicate your bed file in this yaml parameter:
```
coverage: /path/exome.bed
```

To limit variant calling to a given region, indicate your bed file in this yaml parameter:

```
variant_regions: /path/exome.bed
```

If you have issues please ensure the bed file genome version is the same as your are using here, here are some [additional tips](https://bcbio-nextgen.readthedocs.io/en/latest/contents/configuration.html#input-file-preparation).

## Installation of bcbio

### File size estimation

A consideration for using bcbio is memory, these are some estimates for space requirements - most notable is the large amount of space required by the databases which are used for annotation of variants (population, functional, etc.).

data:

|   | depth  | fastq.gz  | bam  | consensus bam  | vcf.gz  | QC metrics  |
|---|---|---|---|---|---|---|
| Exome (50Mb)  | 120x  | 5-10 Gb  | 10-30 Gb  | 5 Gb  | 3-6 Mb  | 6 Mb  |
| AIO (700kb) | 120x  | 300-500 Mb  | 150 Mb  | 75 Mb  | 3-6 Mb  | 6 Mb  |

bcbio:

|   | total  |
|---|---|
| Tools  | 4 Gb  |
| Databases  | ~200 Gb  |


## Installation instructions

Follow the install instructions found on the [bcbio docs](https://bcbio-nextgen.readthedocs.io/en/latest/contents/installation.html). Note that this installation can take several hours.

You can install everything as default with the addition of the gemini databases. As indicated by the `--datatarget gemini` flag. This is required to install databases that will be used for prioritization of somatic variant calls.

```
wget https://raw.github.com/bcbio/bcbio-nextgen/master/scripts/bcbio_nextgen_install.py

python bcbio_nextgen_install.py /usr/local/share/bcbio --tooldir=/usr/local \
        --genomes hg19 --aligners bwa --datatarget gemini
```

## UMI specific steps
### Annotate fastq files with UMI information

```
bcbio_fastq_umi_prep.py autopair -c <cores_to_use> <list> <of> <fastq> <files>
```

This will recognize which fastq file contains the UMI sequences and then annotate your R1 and R2 files with the UMI sequences in the fastq read names.

### Adjust UMI consensus making parameters

Parameters for [fgbio consensus making](http://fulcrumgenomics.github.io/fgbio/tools/latest/CallMolecularConsensusReads.html) can be adjusted in the yaml configuration file:
```
fgbio:
 options: [--min-reads, 2]
```

The default options are as such:
```
--min-reads: 1,
--max-reads: 100000,
--min-map-q: 1,
--min-base-quality: 13,
--min-input-base-quality: 2,
--edits: 1
```

## Regions for analysis

To indicate a region to calculate coverage in QC metrics, indicate your bed file in this yaml parameter:
```
coverage: /path/exome.bed
```

To limit variant calling to a given region, indicate your bed file in this yaml parameter:

```
variant_regions: /path/exome.bed
```

If you have issues please ensure the bed file genome version is the same as your are using here, here are some [additional tips](https://bcbio-nextgen.readthedocs.io/en/latest/contents/configuration.html#input-file-preparation).

## Adjust resources

Resources can be adjusted in the yaml configuration file, under the resources section:

```
resources:
 default:
  memory: 7G
  cores: 7
  jvm_opts: ["-Xms750m", "-Xmx7000m"]
```

Customize this section to utilize the available cores and RAM in your own machine. Note the memory is PER CORE. So in this example here, I am using 7 cores with a total of 49 Gb of RAM.

## Execution

After installation and proper configuration the workflow can be executed using:

```
bcbio_nextgen.py bcbio_pre-cancer.yaml -n <cores_to_use>
```
