# :gem: `metaGEM` Tutorial

## Intro

### Objective
Showcase how the metaGEM workflow can be used to predict metabolic interactions directly from metagenomic data from your own gut microbiome. Unseenbio test kits were sent for sequencing on September 28 & October 21 2020, resulting in two 101 bp paired end reads sets with IDs S2772Nr1 and S2772Nr3. You can find the predicted interactions under the `SMETANA` folder or by clicking [here](https://github.com/franciscozorrilla/unseenbio_metaGEM/blob/main/SMETANA/wgs_S2772Nr3_detailed.tsv). Summary files from intermediate results can be found under the `stats` folder.

For a more advanced tutorial, check out the resources we put together for the [SymbNET: from metagenomics to metabolic interactions](https://www.ebi.ac.uk/training/events/symbnet-2022/) course in 2022.

[![Tutorial2](https://img.shields.io/badge/SymbNET-Tutorial-red)](https://github.com/franciscozorrilla/SymbNET)

### Wiki
Please refer to the [metaGEM wiki](https://github.com/franciscozorrilla/metaGEM/wiki) for further documentation, tips & tricks, and FAQs.

### metaGEM setup

Clone the [metaGEM](https://github.com/franciscozorrilla/metaGEM/) repo and run automated setup script:

```
git clone https://github.com/franciscozorrilla/metaGEM.git
cd metaGEM
bash env_setup.sh
```

Load conda and activate environment:
```
module load anaconda3 # The exact module name will depend on the version installed on your cluster
source activate metagem
```

Please refer to the [setup page in the metaGEM wiki](https://github.com/franciscozorrilla/metaGEM/wiki/Quickstart) for further documentation.

Parser usage:
```
Help: bash metaGEM.sh
Usage: bash metaGEM.sh [-t TASK] [-j NUMBER OF JOBS] [-c ALLOCATED CORES] [-m ALLOCATED GB MEMORY] [-h ALLOCATED HOURS] [-l]
Options:
  -t, --task        Specify task to complete
  -j, --nJobs       Specify number of jobs to run in parallel
  -c, --nCores      Specify number of cores per job
  -m, --mem         Specify memory in GB required for job
  -h, --hours       Specify number of hours to allocated to job runtime
  -l, --local       Run jobs on local machine for non-cluster usage
```

### Hardware
The workflow was executed on the European Molecular Biology Laboratory (EMBL) high performance computing cluster.

## metaGEM workflow

1. Quality filter reads with [fastp](https://github.com/OpenGene/fastp)
2. Assembly with [megahit](https://github.com/voutcn/megahit)
3. Draft bin sets with [CONCOCT](https://github.com/BinPro/CONCOCT),[MaxBin2](https://sourceforge.net/projects/maxbin2/), and [MetaBAT2](https://sourceforge.net/projects/maxbin2/)
4. Refine & reassemble bins with [metaWRAP](https://github.com/bxlab/metaWRAP)
5. Taxonomic assignment with [GTDB-tk](https://github.com/Ecogenomics/GTDBTk)
6. Relative abundances with [bwa](https://github.com/lh3/bwa) and [samtools](https://github.com/samtools/samtools)
7. Reconstruct & evaluate genome-scale metabolic models with [CarveMe](https://github.com/cdanielmachado/carveme) and [memote](https://github.com/opencobra/memote)
8. Species metabolic coupling analysis with [SMETANA](https://github.com/cdanielmachado/smetana)

### 1. Quality filtering short reads with [fastp](https://github.com/OpenGene/fastp)

Organize paired end reads in subdirectories within the `dataset` folder as shown below. metaGEM will read in the sample IDs based on subfolders present in the `dataset` folder and provide these to the Snakefile to use as wildcards for job submissions.

![Screenshot 2020-12-27 at 18 14 41](https://user-images.githubusercontent.com/35606471/103177108-694a9f80-486f-11eb-8291-cc92dd6785db.png)

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
bash metaGEM.sh -t assemblyVis
```
![Screenshot 2020-12-27 at 23 30 25](https://user-images.githubusercontent.com/35606471/103181656-85643600-489b-11eb-85cf-4a64c9d9b6ea.png)

The above barplots show that the assembly of sample wgs_S2772Nr1 contains ~324 million base pairs across ~56 thousand contigs (average of ~5.8 kbp/contig), while sample wgs_S2772Nr3 contains ~172 million base pairs across ~35 thousand contigs (average of ~4.9 kbp/contig). These contigs should bin nicely.

### 3. Generate draft bin sets with [CONCOCT](https://github.com/BinPro/CONCOCT), [MaxBin2](https://sourceforge.net/projects/maxbin2/), and [MetaBAT2](https://sourceforge.net/projects/maxbin2/), with the help of [bwa](https://github.com/lh3/bwa) + [samtools](https://github.com/samtools/samtools)

Using bwa and samtools, cross map each set of paired end reads against each set of assembled contigs to obtain abundances/coverage of contigs across samples. This information will be used by binners to attain best performance. Submit one job per sample, with 24 cores and 120 GB RAM each, with a maximum runtime of 24 hours:

```
bash metaGEM.sh -t crossMapSeries -j 2 -c 24 -m 120 -h 24
```

Note: the old rule `crossMap` was split into `crossMapSeries` and `crossMapParallel`. Running the mappings in series is more straightforward, but may also become prohibitively expensive from a computational resource point of view for datasets with a large number of samples, e.g. N=1000.

Run each of the binners using contig coverage across samples:

```
bash metaGEM.sh -t concoct -j 2 -c 24 -m 80 -h 10
bash metaGEM.sh -t metabat -j 2 -c 24 -m 80 -h 10
bash metaGEM.sh -t maxbin -j 2 -c 24 -m 80 -h 10
```

We will visualize the output of individual binners after refinement and reassembly.

### 4. Generate final bin sets using [metaWRAP](https://github.com/bxlab/metaWRAP) bin_refinement and bin_reassemble

Refine & reassemble bin sets:

```
bash metaGEM.sh -t binRefine -j 2 -c 24 -m 150 -h 24
bash metaGEM.sh -t binReassemble -j 2 -c 24 -m 150 -h 24
```

Visualize binning output:

```
bash metaGEM.sh -t binningVis
```

![Screenshot 2020-12-29 at 13 15 15](https://user-images.githubusercontent.com/35606471/103286338-eb55d800-49d7-11eb-8d38-c10046b708f8.png)

We can see that samples wgs_S2772Nr1 and wgs_S2772Nr1 yield a 88 and 47 refined and reassembled bins respectively, for a total of 135 bins. Although CONCOCT tends to outperform MetaBAT2 and MaxBin2 in terms of number of bins generated (particularly so when there are more samples available to exploit contig coverage information), in this case MetaBAT2 generated the most bins. This highlights the strength and felxibility of the metaGEM binning implementation, where draft bin sets from multiple binners are refined and reassembled to obtain the best possible bins (average completeness 86.5% & average contamination 1.2%) instead of relying on a single binning approach. Indeed, CONCOCT tends to generate bins with higher completion (average completeness 89.2% & average contamination 1.9%) while MetaBAT2 tends to generate bins with lower contamination (average completeness 83.4% & average contamination 1.6%). Although MaxBin2 generated more bins than shown in the figure above, most of them did not meet the medium quality criteria of >50% completeness & <5% contamination due to high contamination. Furthermore, metaWRAP reassembled bins improve the contiguity of bins (average size 2.45 Mbp & average contigs	161.9) compared to CONCOCT (average size 2.45 Mbp	& 287.5 contigs on average) or MetaBAT2 (average size 2.38 Mbp &	167.5 contigs on average).

### 5. Taxonomic assignment with [GTDB-tk](https://github.com/Ecogenomics/GTDBTk)

First lets extract our DNA bins from the metaWRAP reassembly output:

```
bash metGEM.sh -t extractDnaBins
```

Run GTDB-tk for taxonomic classification:

```
bash metaGEM.sh -t gtdbtk -j 2 -c 24 -m 80 -h 12
```
We will visualize taxonomic annotations after calculating relative abundances.

### 6. Relative abundances with [bwa](https://github.com/lh3/bwa) and [samtools](https://github.com/samtools/samtools)

```
bash metaGEM.sh -t abundance -j 2 -c 24 -m 80 -h 12
```

Let's visualize the taxonomy and relative abundances:

```
bash metaGEM.sh -t compositionVis
```

![Screenshot 2020-12-30 at 21 32 09](https://user-images.githubusercontent.com/35606471/103382320-876e0500-4ae6-11eb-857a-236cdb94bca3.png)

Excluded from the figure above are the relative abundances of genomes with undefined species-level taxonomy, accounting for 24.9% and 12.7% of the relative abundances of samples wgs_S2772Nr1 and wgs_S2772Nr3.

We can also do some additional manual analysis in R to check how well correlated the abundances of individual species are between samples:

```
taxab %>% select(user_genome,sample,species,rel_ab) %>% 
  group_by(species) %>% 
  mutate(count=n()) %>% 
  filter(count==2) %>% 
  select(-count,-user_genome) %>% 
  pivot_wider(values_from = rel_ab,names_from = sample) -> scatter_dat

rval=cor(scatter_dat$wgs_S2772Nr1,scatter_dat$wgs_S2772Nr3)
lm_fit = summary(lm(wgs_S2772Nr1~wgs_S2772Nr3,data=scatter_dat))
pval=lm_fit$coefficients[8]

ggplot(scatter_dat,aes(x=wgs_S2772Nr1,y=wgs_S2772Nr3)) +
  geom_point() +
  geom_smooth(method = "lm") +
  annotate("text", label = paste0("Pearson's r = ",signif(rval,digits = 4)), x = 0.015, y = 0.07,size=5) +
  annotate("text", label = paste0("p-value = ",signif(pval,digits = 4)), x = 0.015, y = 0.06,size=5)
```

![Screenshot 2020-12-31 at 15 37 10](https://user-images.githubusercontent.com/35606471/103416327-124e0e80-4b7e-11eb-885f-8d7ef24cb3ad.png)

### 7. Reconstruct and evaluate genome scale metabolic models using [CarveMe](https://github.com/cdanielmachado/carveme) and [memote](https://github.com/opencobra/memote)

Although most of the CarveMe dependencies are installed in the metaGEM conda environment using the metaGEM conda recipie, the CPLEX solver requires users to register with IBM to obtain a [free academic license](https://community.ibm.com/community/user/datascience/blogs/xavier-nodet1/2020/07/09/cplex-free-for-students).

Let's extract the ORF annotated protein bins from the metaWRAP reassembly output and dump them into a single folder for easy parallel job submission:

```
bash metaGEM.sh -t extractProteinBins
```

The models will be gapfilled on complete media by default, but this can be easily tweaked in the `config.yaml` file. It is also possible to add custom media recipies to the `media_db.tsv` file, which uses BiGG database metabolite IDs. Let's now run CarveMe on the generated protein bins. Note that metaGEM will read MAG IDs from the `protein_bins` folder,i.e. the location where the  `extractProteinBins` rule deposits the protein bin files. In this case we will submit 135 jobs (one per MAG) with 4 cores and 20 GB RAM each, and a maximum runtime of 4 hours:

```
bash metaGEM.sh -t carveme -j 135 -c 4 -m 20 -h 4
```

After the models are generated we can evaluate them using memote:

```
bash metaGEM.sh -t memote -j 135 -c 4 -m 20 -h 2
```

Memote generates detailed reports for each model, which can also be summarized and analyzed using the [script](https://github.com/biosustain/memote-meta-study) `cli.py`. We can quickly visualize the number of metabolites, genes, and reactions of the models with metaGEM:

```
bash metaGEM.sh -t modelVis
```

![Screenshot 2020-12-31 at 18 28 11](https://user-images.githubusercontent.com/35606471/103421830-4b927880-4b96-11eb-88f8-297588eb866a.png)


The models have similar distributions of metabolites, reactions, and genes as compared to other models in our [publication](https://doi.org/10.1101/2020.12.31.424982 ).

![Screenshot 2021-01-06 at 18 43 27](https://user-images.githubusercontent.com/35606471/103807997-3793be80-504f-11eb-9b0c-bfb895fd32ba.png)


### 8. Community simulations with [SMETANA](https://github.com/cdanielmachado/smetana)

First let's organize our models into sample specific sub-directories for easy job submission:

```
bash metaGEM.sh -t organizeGEMs
```

The SMETANA algorithm calculates species coupling scores (SCS), metabolite uptake scores (MUS), and metabolite production scores (MPS), which are multiplied to obtain a SMETANA score for each possible interaction between every species pair. For more information about the implementation and interpretation of SMETANA simulations please refer to the [original paper](https://www.pnas.org/content/112/20/6449).

We can easily configure simulation media for computational experiments by modifying the `config.yaml` file. By default, metaGEM will simulate the communities in all media present in the base `media_db.tsv` file. Again, one could create custom simulation media by expanding the `media_db.tsv` file based on BiGG database metabolite IDs. Beware of the fact that large communities may require long runtimes to complete, especially if many simulation media are provided (community simulations are run per media in series). 

In this demo we will set up the same computational experiment as in the [metaGEM paper](https://www.biorxiv.org/content/10.1101/2020.12.31.424982v2.full), where the models are gapfilled on complete media (M3) and simulated in media without aromatic amino acids (M11). The media compositions were provided by the authors of [this paper](https://www.nature.com/articles/s41564-018-0123-9), and are summarized in subfigure 1d:

![](https://user-images.githubusercontent.com/35606471/105162180-2f855580-5b0a-11eb-824b-5b30f10d66cd.png)

```
bash metaGEM.sh -t smetana -j 2 -c 24 -m 40 -h 24
```

The SMETANA output file contains predicted metabolic interactions between members of the simulated community:

```
community	medium	receiver	donor	compound	scs	mus	mps	smetana
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.33.p	M_acald_e	0.0425531914893617	0.12	1	0.005106382978723404
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.33.p	M_ala__D_e	0.0425531914893617	0.23	1	0.009787234042553192
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.33.p	M_alaala_e	0.0425531914893617	0.02	1	0.000851063829787234
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.33.p	M_phe__L_e	0.0425531914893617	1.0	1	0.0425531914893617
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.19.s	M_acald_e	0.02127659574468085	0.12	1	0.002553191489361702
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.19.s	M_ala__D_e	0.02127659574468085	0.23	1	0.004893617021276596
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.19.s	M_alaala_e	0.02127659574468085	0.02	1	0.000425531914893617
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.40.s	M_acald_e	0.02127659574468085	0.12	1	0.002553191489361702
all	M11	wgs_S2772Nr3_bin.50.p	wgs_S2772Nr3_bin.40.s	M_ala__D_e	0.02127659574468085	0.23	1	0.004893617021276596
```

Let's visualize the SMETANA simulation results from sample wgs_S2772Nr3. We can start by plotting boxplots of the most commonly exchanged compounds within the community:

![smetana_plot1](https://user-images.githubusercontent.com/35606471/107069103-5d8cba00-67d9-11eb-8950-d7a6ee7fbd77.png)

As expected, we see that tryptophan, tyrosine, and phenylalanine are exchanged within this community of gut microbes. We can also group SMETANA scores by donor and receiver to identify keystone species:

![don_rec_plot](https://user-images.githubusercontent.com/35606471/107070062-b0b33c80-67da-11eb-9a6d-77b6002f7453.png)

The top subplot in the figure above suggests that *Parasutterella excrementihominis* and *Agathobaculum butyriciproducens* are key donor species, even though they account for 0.61% and 0.97% of the community relative abundance, respectively. On the other hand, the bottom subplot suggests that *Faecalicatena torques* and *Bacteroides salyersiae* are the most heavily dependant on other community members. Interestingly, we also see that *Faecalibacterium prausnitzii K*, the most abundant species in the community, is completely dependant on *Agathobaculum butyriciproducens* for hydrogen sulfide (i.e. SMETANA score equals 1 for this interaction).

## Please cite

```
Francisco Zorrilla, Filip Buric, Kiran R Patil, Aleksej Zelezniak
metaGEM: reconstruction of genome scale metabolic models directly from metagenomes
Nucleic Acids Research, Volume 49, Issue 21, 2 December 2021, Page e126, https://doi.org/10.1093/nar/gkab815
```
