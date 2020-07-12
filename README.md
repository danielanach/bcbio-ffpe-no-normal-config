# bcbio-ffpe-no-normal-config

Configuration to use [bcbio workflow](https://bcbio-nextgen.readthedocs.io/en/latest/contents/pipelines.html#cancer-variant-calling) with the intention of obtaining somatic, germline and copy number variant calling from cancer / ffpe-no-normal FFPE samples without matched normals.

This configuration assumes:
* Targeted capture library sequencing preparation (e.g. Agilent SureSelect XTHS)
* Paired-end sequencing
* If UMIs were used, they are contained in separate fastq file
* Samples are 'tumor/lesion' without matched normal



This workflow leverages a population database strategy for filtering of germline variants described in the [bcbio documentation](https://bcbio-nextgen.readthedocs.io/en/latest/contents/pipelines.html#cancer-variant-calling) as well as in this [benchmarking blog post](http://bcb.io/2015/03/05/cancerval/). This somatic variant prioritization approach is the biggest benefit of utilizing the bcbio cancer variant calling workflow.

**Note on pool of normal:**
The best practice for tumor-only, or paired analysis is to include a panel of normal (PON) samples that were processed in the same manner as your samples during variant calling in order to remove techinical artifacts. If this is unavailable to you, then a publicly available PON is still better than variant calling without it. We have previously used this publicly available PON and the sensitivity / precision was improved with its usage: https://console.cloud.google.com/storage/browser/details/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf. 

**Note on UMIs:**
This workflow can also use unique molecular identifiers (UMIs) for additional error correction and PCR duplicate removal. However, it is not currently recommended for somatic variant calling with this configuration since the variant callers used here are not UMI-aware and the base quality scores are altered in consensus bam. But you can use the CNV calls and the output bam file.   


## Configuration file

The pipeline is executed using a yaml file and in this repository the example is named: `bcbio_ffpe-no-normal.yaml`.

The yaml file is edited for each workflow execution to reflect necessary parameters for that analysis.

### Adjust sample information

At the minimum you will need to edit information about your samples:
```
- description: sample1
  files: [/path/sample1_R1.fq.gz, /path/to/sample1_R2.fq.gz]
  metadata:
    batch: [sample1]
    phenotype: tumor
```

Unless you have matched normal, then do not change the phenotype for your samples.

### Adjust regions for analysis

To specify a list of targeted regions to calculate coverage in QC metrics, indicate your bed file location in this yaml parameter:
```
coverage: /path/exome.bed
```

To limit variant calling to a given targeted region, indicate your bed file in this yaml parameter:

```
variant_regions: /path/exome.bed
```

If you have issues please ensure the bed file genome version (e.g. hg19, hg38, or GRch38) is the same as the one used for read alignment, here are some [additional tips](https://bcbio-nextgen.readthedocs.io/en/latest/contents/configuration.html#input-file-preparation).

If you have WGS data, you can remove these two parameters from the yaml file.

### Adjust variant calling parameters

Parameters for variant calling can be adjusted in the yaml configuration file.

Indicate which variant callers you would like to use here:

```
variantcaller: [vardict, mutect2]
```

Indicate the number of variant callers which must call a variant in order to be reported:

```
ensemble:
  numpass: 2
```

Low variant allelic frequency (VAF) variants are more likely to contain false variants which are artifacts of DNA damage, especially common is C>T in FFPE samples.

Adjust your minimum VAF in this line of the yaml configuration file. Our common practice is to leave it at 0 for variant calling, then filter for VAF > 0.1 for downstream analysis. This enables us to better assess the levels of damage artifacts in our samples prior to removing them.

```
min_allele_fraction: 0
```

### Adjust UMI consensus making parameters

Parameters for [fgbio consensus making](http://fulcrumgenomics.github.io/fgbio/tools/latest/CallMolecularConsensusReads.html) can be adjusted in the yaml configuration file, the only parameter adjusted in this configuration is to use a minimum of 2 reads to create a consensus read. This is indicated in the configuration file via this portion of the yaml file.
```
fgbio:
 options: [--min-reads, 2]
```

### Adjust resources

Prior to execution of the pipeline, edit the yaml configuration file resources section:

```
resources:
 default:
  memory: 7G
  cores: 7
  jvm_opts: ["-Xms750m", "-Xmx7000m"]
```

Customize this section to utilize the available cores and RAM in your own machine. Note the memory is PER CORE. So in this example here, I am using 7 cores with a total of 49 Gb of RAM.

## Other notes:


## Installation of bcbio

### File size estimation

A consideration for using bcbio is memory, these are some estimates for space requirements - most notable is the large amount of space required by the databases which are used for annotation of variants (population, functional, etc.).

**data:**

|   | depth  | fastq.gz  | bam  | consensus bam  | vcf.gz  | QC metrics  |
|---|---|---|---|---|---|---|
| Exome (50Mb)  | 120x  | 5-10 Gb  | 10-30 Gb  | 5 Gb  | 3-6 Mb  | 6 Mb  |
| AIO (700kb) | 120x  | 300-500 Mb  | 150 Mb  | 75 Mb  | 3-6 Mb  | 6 Mb  |

**bcbio:**

|   | total  |
|---|---|
| Tools  | 4 Gb  |
| Databases  | ~90 Gb  |


### Installation instructions

Follow the install instructions found on the [bcbio docs](https://bcbio-nextgen.readthedocs.io/en/latest/contents/installation.html). Note that this installation can take several hours.

You can install everything as default with the addition of the gemini databases. As indicated by the `--datatarget gemini` flag. This is required to install databases that will be used for prioritization of somatic variant calls.


**Example:**
```
wget https://raw.github.com/bcbio/bcbio-nextgen/master/scripts/bcbio_nextgen_install.py

python bcbio_nextgen_install.py /usr/local/share/bcbio --tooldir=/usr/local \
        --genomes hg19 --aligners bwa --datatarget gemini
```

## Fastq pre-processing for UMIs

If your UMI reads are contained in a third fastq files, you will need to annotate your R1 and R2 fastq read names to contain the associated UMIs.

```
bcbio_fastq_umi_prep.py autopair -c <cores_to_use> <list> <of> <fastq> <files>
```

This assumes you have R1, R2, and UMI fastq files. This will recognize which fastq file contains the UMI sequences and then annotate your R1 and R2 files with the UMI sequences in the fastq read names.

## Execution

After bcbio installation, proper yaml file configuration, and UMI pre-processing, copy your yaml file into your working directory.

The workflow can be executed using:

```
bcbio_nextgen.py bcbio_ffpe-no-normal.yaml -n <cores_to_use>
```

## Results

The final output structure will contain several folders and files.

```
/path/project_name      
│── project_name       
│       └── sample_1-ensemble-annotated.vcf.gz   # Somatic variant calls - (intersection mutect2 and vardict)
│       └── sample_2-ensemble-annotated.vcf.gz   # Somatic variant calls - (intersection mutect2 and vardict)
│       └── multiqc      
│               └── etc.   
│── sample_1    
│       └── sample_1-mutect2-germline.vcf.gz  # Germline variant calls - (mutect2)
│       └── sample_1-vardict-germline.vcf.gz    # Germline variant calls - (vardict)
│       └── sample_1-cnvkit.cns                 # CNA segments
│       └── sample_1-cnvkit-call.cns            # CNA segment calls
│       └── etc.
└──  sample_2     
        └── sample_2-mutect2-germline.vcf.gz  # Germline variant calls - (mutect2)
        └── sample_2-vardict-germline.vcf.gz    # Germline variant calls - (vardict)
        └── sample_2-cnvkit.cns                 # CNA segments
        └── sample_2-cnvkit-call.cns            # CNA segment calls
        └── etc.
```

The final somatic VCF will be contained in the main project folder with the extension \*-ensemble-annotated.vcf as well as multiQC metrics. Germline VCF and copy number variants will be contained in individual sample folders.

Note there will be many other intermediate files and QC metrics that may or may not be useful to you.
