# bcbio-pre-cancer-config

Configuration to use bcbio workflow for pre-cancer tumor only ffpe samples for somatic variant calls.

Method assumes a targeted hybridization library sequencing preparation such as Agilent XTHS.

This workflow leverages unique molecular identifiers (UMIs) for additional error correction and PCR duplicate removal. It also uses the population database approach of filtering germline samples described in the [bcbio documentation](https://bcbio-nextgen.readthedocs.io/en/latest/contents/pipelines.html#cancer-variant-calling) as well as in this [benchmarking blog post](https://bcbio-nextgen.readthedocs.io/en/latest/contents/pipelines.html#cancer-variant-calling)



## Installation

Follow the install instructions found on the [bcbio docs](https://bcbio-nextgen.readthedocs.io/en/latest/contents/installation.html).

Additionally include the following flag in your install to download the gemini database.
```
--datatarget gemini
```

## UMI specific steps
### Annotate fastq files with UMI information

```
bcbio_fastq_umi_prep.py autopair -c <cores_to_use> <list> <of> <fastq> <files>
```

This will recognize which fastq file contains the UMI sequences and then annotate your R1 and R2 files with the UMI sequences in the fastq read names.

### Adjust minimum number of reads per UMI

Parameters for [fgbio consensus making](http://fulcrumgenomics.github.io/fgbio/tools/latest/CallMolecularConsensusReads.html) can be adjusted in the .yaml configuration file:
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

## Adjust resources

Resources can be adjusted in the .yaml configuration file, under the resources section:

```
resources:
 default:
  memory: 7G
  cores: 7
  jvm_opts: ["-Xms750m", "-Xmx7000m"]
```

Customize this section to utilize the available cores and RAM in your own machine. Note the memory is PER CORE. So in this examble here, I am using 7 cores with a total of 49 Gb of RAM.

## Execution

After installation and proper configuration the workflow can be executed using:

```
bcbio_nextgen.py bcbio_pre-cancer.yaml -n <cores_to_use>
```
