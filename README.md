# hiv-rna-velocity
RNA velocity analysis for samples from model of HIV latency

## Background:

A paper from a few years ago (PMID: 30282021) reported a scRNAseq dataset (10x Chromium V2) collected from HIV infected and uninfected cells from an in vitro model of HIV latency. The goal of the original paper (as is my goal here) was to identify potential mechanisms that lead to the HIV latent reservoir. If these mechanisms can be disrupted (ie via a new class of drugs) then HIV stands a chance of being cured permanently, not just suppressed with drugs.

The paper was published around the time the first RNA velocity paper was published (PMID: 30089906). As such, it is unlikely the authors considered using this analysis technique which might help to give a more detailed understanding of the latent HIV reservoir. 

In this analysis I have used the excellent scVelo package (w/ scanpy and anndata) to estimate RNA velocity components (https://doi.org/10.1101/820936). The RNA velocity approach is based on the idea that unspliced transcripts can be used to estimate the "future state" of the cell (and thus the local trajectory or velocity of each cell). These approaches require that both the unspliced and spliced transcripts be enumerated during mapping. 

To get the necessary setup to apply this approach I mapped the authors raw fastq data with salmon alevin (PMID: 30917859) on an EC2 instance. For building the index I used the GRCh38 annotated transcripts (from GENCODE) and appended the HIV genome (NL43-D6-dreGFP). I also added a matching "unspliced" transcript to each spliced transcript in the human genome using the getFeatureRanges function from the eisaR package. Inclusion of the intronic regions blew up the size of my salmon index by a lot and I ended up needing to use an EC2 instance with more than ~50GB of memory. I chose r5.xlarge to build the index followed by c5.12xlarge to do the mapping. Mapping took a little under an hour per sample.

## Results
### Combined sample analysis
First I merged all the samples and plotted the UMAP embedding to get an overall view of data. The source of the cells was not used for clustering. There are obvious batch effects:
![Groups](/figures/umap_merged_group.png)
![Merged HIV](/figures/umap_merged_CD3d.png)

It is reassuring to see that samples which were not HIV infected (d1u, d2u) did not have substantial expression of HIV. The following plot of CD3 expression is more interesting:

![Merged CD3d](/figures/umap_merged_CD3d.png)

This plot clearly show a sub-population of cells that lack CD3d expression in donor 1 (but not donor 2). CD3 is a marker of the T-cell lineage which strongly suggests that this population of CD3- cells are not T-cells. It is likely that there is some type of contaminating population - possibly the H80 feeder cells used to stimulate the T-cells.

Now I plot the expression of Beta-2-microglobulin (B2M):
![Merged B2M](/figures/umap_merged_B2M.png)

This shows that in d1u, d2u, and d2i that there is a substantial fraction of cells with very low expression of B2M. This could be significant as B2M is required to produce MHC class 1 surface moleculars which are used by CD8+ T-cells to kill infected cells.

### Individual sample analysis
Now I will examine each sample separately using the RNA-velocity. For now, I am recalculating the UMAP embedding for each sample (therefore individual samples are not directly comparable across plots). I am still working on transferring the merged embedding (after batch correction) to each individual sample which will make samples easier to compare.

#### Donor 1, infected
![B2M d1i](/figures/scvelo_d1i_B2M_umap.png)

#### Donor 2, infected
![B2M d2i](/figures/scvelo_d2i_B2M_umap.png)

#### Donor 1, uninfected
![B2M d1u](/figures/scvelo_d1u_B2M_umap.png)

#### Donor 2, uninfected
![B2M d2u](/figures/scvelo_d2u_B2M_umap.png)



## Conclusions:
It is clear so far from the analysis that there is contamination of the CD4 T-cells with non-T cells in donor 1 (but not donor 2). My hypothesis is that these are likely the feeder cells (H80) which are used to keep the immune cells activated and alive. 

Second, it is clear that there is a population of cells that lack beta-2-microglobulin (B2M) expression but express CD3. This subpopulation can be clearly seen in figures scvelo\_d1u\_B2M\_umap.png, scvelo\_d2u\_B2M\_umap.png, scvelo\_B2M\_CD3d\_umap.png (shown above). If this turns out to be real it could suggest that a subset of Tcells can downregulate B2M - suggesting a possible mechanism by which T-cells can survive for long periods of time by avoiding CTLs (which require expression of B2M as part of MHC-1 in order to mediate killing). I am looking into what other genes correlate with B2M expression in this dataset in hopes of a hint for what might be causing loss of B2M.

## What this code does:
The notebook extracts the data from the alevin binary format (in R), separates-out the "unspliced" and "spliced" counts using splitSE (from tximeta), and then converts the data into a format to be used by scVelo (anndata) in Python. I then remove droplets with very low and very high counts and select only genes with high dispersion across all the samples which will be used for velocity estimation.

From here, various processing calculations are performed including normalization. Data is stored along the way. Plots are generated.

## How to run
```
git clone wmatern/hiv-rna-velocity
```
Go to this [link](https://drive.google.com/file/d/1Auur5sKTyfjveIyXf8dpGRbu5BCeMQKw/view?usp=sharing) and download the entire zip file containing the input data (alevin mapping results).

Place the file into the hiv-rna-velocity/ directory.

```
cd hiv-rna-velocity/
unzip input.zip
conda env create -f environment.yml
conda activate wmatern_hiv
```

And then either:
```
jupyter notebook analysis.ipynb
```
or
```
jupyter-lab analysis.ipynb
```

The docker image I used to create input fasta files (spliced + unspliced transcriptome) from genome files is on dockerhub:
```
docker pull wmatern/alevin_scrnaseq_velocity:0.1
```
The image I used to build the salmon index and map reads:
```
docker pull combinelab/salmon
```

## Future work:
1. Mitochodrial gene expression.
2. Identify genes that correlate with B2M.
3. Adjust RNA velocity fitting params to clean-up plots.

## Acknowledgements:
T Bradley, G Ferrari, BF Haynes, DM Margolis, EP Browne. Single-cell analysis of quiescent HIV infection reveals host transcriptional profiles that regulate proviral latency. Cell reports 25 (1), 107-117. https://doi.org/10.1016/j.celrep.2018.09.020.

[This](https://combine-lab.github.io/alevin-tutorial/2020/alevin-velocity/) tutorial on using alevin with RNA velocity.
