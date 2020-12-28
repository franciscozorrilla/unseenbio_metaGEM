# [metaGEM](https://github.com/franciscozorrilla/metaGEM/) workflow applied to samples from [unseen bio](https://unseenbio.com/)

### Objective
Showcase how the metaGEM workflow can be used to explore the human gut microbiome using whole metagenome shotgun sequencing data obtained from unseen bio's at-home test kits. Unseenbio test kits were sent for sequencing on September 28 & October 21 2020, resulting in two 101 bp paired end reads sets with IDs S2772Nr1 and S2772Nr3.

### metaGEM workflow

0. metaGEM setup
1. Quality filter reads (fastp)
2. Assembly (megahit)
3. Draft bin sets (CONCOCT,MaxBin2,MetaBAT2)
4. Refine & reassemble bins (metaWRAP)
5. Reconstruct genome-scale metabolic models (CarveMe)
6. Species metabolic coupling analysis (SMETANA)
7. Taxonomic assignment(GTDB-tk)
8. Relative abundances (bwa)
9. Pangenome analysis (roary)
10. Growth rate estimation (GRiD)
11. Eukaryotic draft bins (EukRep,EukCC)

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

Organize paired end reads in subdirectories as shown below.

![Screenshot 2020-12-27 at 18 14 41](https://user-images.githubusercontent.com/35606471/103177108-694a9f80-486f-11eb-8291-cc92dd6785db.png)

### 1. Quality filtering short reads with fastp

Submit one quality filtering job per sample, with 2 cores and 20 GB RAM each, with a maximum runtime of 2 hours:
```
bash metaGEM.sh -t fastp -j 2 -c 2 -m 20 -h 2
```

Visualize quality filtering results:
```
bash metaGEM.sh -t qfilterVis
```
![Screenshot 2020-12-27 at 18 44 38](https://user-images.githubusercontent.com/35606471/103177571-9dc05a80-4873-11eb-9f22-2972abff3081.png)

### 2. Assembly of short reads with megahit

Submit one assembly job per sample, with 48 cores and 250 GB RAM each, with a maximum runtime of 100 hours:
```
bash metaGEM.sh -t megahit -j 2 -c 48 -m 250 -h 100
```

Visualize assembly results:
```
bash metaGEM.sh -t qfilterVis
```
![Screenshot 2020-12-27 at 23 30 25](https://user-images.githubusercontent.com/35606471/103181656-85643600-489b-11eb-85cf-4a64c9d9b6ea.png)


### 3. Generate draft bin sets with CONCOCT, MaxBin2, and MetaBAT2, with the help of bwa + samtools

Using bwa and samtools, cross map each set of paired end reads against each set of assembled contigs to obtain abundances/coverage of contigs across samples. This information will be used by binners to attain best performance:

```
bash metaGEM.sh -t crossMap -j 2 -c 48 -m 250 -h 100
```

Run each of the binners using contig coverage across samples:

```
bash metaGEM.sh -t concoct -j 2 -c 24 -m 100 -h 10
bash metaGEM.sh -t metabat -j 2 -c 24 -m 100 -h 10
bash metaGEM.sh -t maxbin -j 2 -c 24 -m 100 -h 10
```

### 4. Generate final bin sets using [metaWRAP](https://github.com/bxlab/metaWRAP) bin_refinement and bin_reassemble

It is recommended to install metaWRAP in its own isolated conda environment. This can be done using conda install or based on the metaWRAP.yml conda recipie file found in the metaGEM repo:

```
conda env create -f metaWRAP_env.yml
source activate metawrap
```

Refine & reassembled bin sets:

```
bash metaGEM.sh -t binRefine -j 2 -c 48 -m 250 -h 100
bash metaGEM.sh -t binReassemble -j 2 -c 48 -m 250 -h 100
```

### 5. Reconstruct genome scale metabolic models using CarveMe

Run CarveMe:

```
bash metaGEM.sh -t carveme -j 2 -c 2 -m 4 -h 2
```

Evaluate models using memote:

```
bash metaGEM.sh -t memote -j 2 -c 4 -m 8 -h 2
```

