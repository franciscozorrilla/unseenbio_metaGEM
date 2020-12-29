# [metaGEM](https://github.com/franciscozorrilla/metaGEM/) workflow applied to samples from [unseen bio](https://unseenbio.com/)

### Objective
Showcase how the metaGEM workflow can be used to explore the human gut microbiome using whole metagenome shotgun sequencing data obtained from unseen bio's at-home test kits. Unseenbio test kits were sent for sequencing on September 28 & October 21 2020, resulting in two 101 bp paired end reads sets with IDs S2772Nr1 and S2772Nr3.

### metaGEM workflow

0. metaGEM setup - [x]
1. Quality filter reads with [fastp](https://github.com/OpenGene/fastp) - [x]
2. Assembly with [megahit](https://github.com/voutcn/megahit) - [x]
3. Draft bin sets with [CONCOCT](https://github.com/BinPro/CONCOCT),[MaxBin2](https://sourceforge.net/projects/maxbin2/), and [MetaBAT2](https://sourceforge.net/projects/maxbin2/) - [x]
4. Refine & reassemble bins with [metaWRAP](https://github.com/bxlab/metaWRAP) - [x]
5. Reconstruct & evaluate genome-scale metabolic models with [CarveMe](https://github.com/cdanielmachado/carveme) and [memote](https://github.com/opencobra/memote) - [in progress]
6. Species metabolic coupling analysis with [SMETANA](https://github.com/cdanielmachado/smetana) - [ ]
7. Taxonomic assignment with [GTDB-tk](https://github.com/Ecogenomics/GTDBTk) - [ ]
8. Relative abundances with [bwa](https://github.com/lh3/bwa) and [samtools](https://github.com/samtools/samtools) - [ ]
9. Pangenome analysis with [roary](https://github.com/sanger-pathogens/Roary) - [ ]
10. Growth rate estimation with [GRiD](https://github.com/ohlab/GRiD) - [ ]
11. Eukaryotic draft bins with [EukRep](https://github.com/patrickwest/EukRep) and [EukCC](https://github.com/Finn-Lab/EukCC) - [ ]

### Hardware
The workflow was executed on the European Molecular Biology Laboratory (EMBL) high performance computing cluster.

### 0. metaGEM setup

Obtain 5 main helper files from the [metaGEM](https://github.com/franciscozorrilla/metaGEM/) repo:
* metaGEM_env.yml: conda environment recipie.
* Snakefile: contains metaGEM workflow instructions.
* metaGEM.sh: parser used for easier user interface with Snakefile.
* config.yaml: file used for set up and tweaking pipeline parameters.
* cluster_config.json: file used for job submissions to cluster.

Parser usage:
```
Help: bash metaGEM.sh
Usage: bash metaGEM.sh [-t TASK] [-j NUMBER OF JOBS] [-c ALLOCATED CORES] [-m ALLOCATED GB MEMORY] [-h ALLOCATED HOURS]
Options:
  -t, --task        Specify task to complete
  -j, --nJobs       Specify number of jobs to run in parallel
  -c, --nCores      Specify number of cores per job
  -m, --mem         Specify memory in GB required for job
  -h, --hours       Specify number of hours to allocated to job runtime
```

Create conda environment and install tools
```
module load Anaconda3
conda env create -f metaGEM.yml
```

Load conda and activate environment:
```
source activate metaGEM
```

Organize paired end reads in subdirectories within the `dataset` folder as shown below. metaGEM will read in the sample IDs based on subfolders present in the `dataset` folder and provide these to the Snakefile to use as wildcards for job submissions.

![Screenshot 2020-12-27 at 18 14 41](https://user-images.githubusercontent.com/35606471/103177108-694a9f80-486f-11eb-8291-cc92dd6785db.png)

### 1. Quality filtering short reads with [fastp](https://github.com/OpenGene/fastp)

Submit one quality filtering job per sample, with 2 cores and 20 GB RAM each, with a maximum runtime of 2 hours:
```
bash metaGEM.sh -t fastp -j 2 -c 2 -m 20 -h 2
```

Visualize quality filtering results:
```
bash metaGEM.sh -t qfilterVis
```
![Screenshot 2020-12-28 at 14 55 37](https://user-images.githubusercontent.com/35606471/103222817-cb151300-491c-11eb-94bc-1f616d90f21d.png)

The above barplot shows that sample wgs_S2772Nr1 has ~4 billions base pairs, while sample wgs_S2772Nr3 has ~8 billion. Both samples have good quality scores, with approximately 98% of all reads/basepairs being retained after quality fitlering.

### 2. Assembly of short reads with [megahit](https://github.com/voutcn/megahit)

Submit one assembly job per sample, with 24 cores and 120 GB RAM each, with a maximum runtime of 24 hours:
```
bash metaGEM.sh -t megahit -j 2 -c 24 -m 120 -h 24
```

Visualize assembly results:
```
bash metaGEM.sh -t qfilterVis
```
![Screenshot 2020-12-27 at 23 30 25](https://user-images.githubusercontent.com/35606471/103181656-85643600-489b-11eb-85cf-4a64c9d9b6ea.png)

The above barplots show that the assembly of sample wgs_S2772Nr1 contains ~324 million base pairs across ~56 thousand contigs (average of ~5.8 kbp/contig), while sample wgs_S2772Nr3 contains ~172 million base pairs across ~35 thousand contigs (average of ~4.9 kbp/contig). These contigs should bin nicely.

### 3. Generate draft bin sets with [CONCOCT](https://github.com/BinPro/CONCOCT), [MaxBin2](https://sourceforge.net/projects/maxbin2/), and [MetaBAT2](https://sourceforge.net/projects/maxbin2/), with the help of [bwa](https://github.com/lh3/bwa) + [samtools](https://github.com/samtools/samtools)

Using bwa and samtools, cross map each set of paired end reads against each set of assembled contigs to obtain abundances/coverage of contigs across samples. This information will be used by binners to attain best performance. Submit one job per sample, with 24 cores and 120 GB RAM each, with a maximum runtime of 24 hours:

```
bash metaGEM.sh -t crossMap -j 2 -c 24 -m 120 -h 24
```

Run each of the binners using contig coverage across samples:

```
bash metaGEM.sh -t concoct -j 2 -c 24 -m 80 -h 10
bash metaGEM.sh -t metabat -j 2 -c 24 -m 80 -h 10
bash metaGEM.sh -t maxbin -j 2 -c 24 -m 80 -h 10
```

We will visualize the output of individual binners after refinement and reassembly.

### 4. Generate final bin sets using [metaWRAP](https://github.com/bxlab/metaWRAP) bin_refinement and bin_reassemble

It is recommended to install metaWRAP in its own isolated conda environment. This can be done using conda install or based on the metaWRAP.yml conda recipie file found in the metaGEM repo:

```
# Install directly from conda
conda create --name metawrap-env --channel ursky metawrap-mg=1.3.2

# Alternatively install from conda recipie 
conda env create -f metaWRAP_env.yml

source activate metawrap-env
```

Refine & reassembled bin sets:

```
bash metaGEM.sh -t binRefine -j 2 -c 24 -m 150 -h 24
bash metaGEM.sh -t binReassemble -j 2 -c 24 -m 150 -h 24
```

Visualize binning output:

```
bash metaGEM.sh -t binningVis
```

![Screenshot 2020-12-29 at 13 15 15](https://user-images.githubusercontent.com/35606471/103286338-eb55d800-49d7-11eb-8d38-c10046b708f8.png)

The above figure shows that sample wgs_S2772Nr1 yielded a total of 88 refined and reassembled bins, while sample wgs_S2772Nr1 yielded 47. Although CONCOCT tends to outperform MetaBAT2 and MaxBin2 in terms of number of bins generated (particularly so when there are more samples available to exploit contig coverage information), in this case MetaBAT2 generated the most bins. This highlights the strength and felxibility of the metaGEM binning implementation, where draft bin sets from multiple binners are refined and reassembled to obtain the best possible bins (average completeness 86.5% & average contamination 1.2%) instead of relying on a single binning approach. Indeed, CONCOCT tends to generate bins with higher completion (average completeness 89.2% & average contamination 1.9%) while MetaBAT2 tends to generate bins with lower contamination (average completeness 83.4% & average contamination 1.6%). Although MaxBin2 generated more bins than shown in the figure above, most of them did not meet the medium quality criteria of >50% completeness & <5% contamination due to high contamination. Furthermore, metaWRAP reassembled bins improve the contiguity of bins (average size 2.45 Mbp & average contigs	161.9) compared to CONCOCT (2.45 Mbp	& 287.5 contigs) or MetaBAT2 (2.38 Mbp &	167.5 contigs).

### 5. Reconstruct and evaluate genome scale metabolic models using [CarveMe](https://github.com/cdanielmachado/carveme) and [memote](https://github.com/opencobra/memote)

Although most of the CarveMe dependencies are installed in the metaGEM conda environment using the metaGEM conda recipie, the CPLEX solver requires users to register with IBM to obtain a [free academic license](https://community.ibm.com/community/user/datascience/blogs/xavier-nodet1/2020/07/09/cplex-free-for-students).

Let's extract the ORF annotated protein bins from the metaWRAP reassembly output and dump them into a single folder for easy parallel job submission:

```
bash metaGEM.sh -t extractProteinBins
```

The models will be gapfilled on complete media by default, but this can be easily tweaked in the config.yaml file. Let's run CarveMe on the generated protein bins. Note that metaGEM will read MAG IDs from the protein_bins folder,i.e. the location where extractProteinBins deposits the protein bins. In this case we will submit 135 jobs (one per MAG) with 4 cores and 20 GB RAM each, and a maximum runtime of 4 hours:

```
bash metaGEM.sh -t carveme -j 135 -c 4 -m 20 -h 4
```

After the models are generated we can evaluate them using memote:

```
bash metaGEM.sh -t memote -j 2 -c 4 -m 8 -h 2
```

