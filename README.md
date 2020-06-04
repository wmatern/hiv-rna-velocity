# hiv-rna-velocity
RNA velocity analysis for samples from model of HIV latency

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

## Background:
A paper from a few years ago (PMID: 30282021) reported a scRNAseq dataset (10x Chromium V2) collected from HIV infected and uninfected cells from an in vitro model of HIV latency. The goal of the original paper (as is my goal here) was to identify potential mechanisms that lead to the HIV latent reservoir. If that could be done (ie via a new class of drugs) then HIV stands a chance of being cured permanently, not just suppressed with drugs.

The paper was published around the time the first RNA velocity paper was published (PMID: 30089906). As such, it is unlikely the authors considered using this analysis technique which might help to give a more detailed understanding of the latent HIV reservoir. 

In this analysis I have used the excellent scVelo package (w/ scanpy and anndata) to estimate RNA velocity components (https://doi.org/10.1101/820936). The RNA velocity approach is based on the idea that unspliced transcripts can be used to estimate the "future state" of the cell (and thus the local trajectory or velocity of each cell). These approaches require that both the unspliced and spliced transcripts be enumerated during mapping. 

To get the necessary setup to apply this approach I mapped the authors raw fastq data with salmon alevin (PMID: 30917859) on an EC2 instance. For building the index I used the GRCh38 annotated transcripts (from GENCODE) and appended the HIV genome (NL43-D6-dreGFP). I also added a matching "unspliced" transcript to each spliced transcript in the human genome using the getFeatureRanges function from the eisaR package. Inclusion of the intronic regions blew up the size of my salmon index by a lot and I ended up needing to use an EC2 instance with more than ~50GB of memory. I chose r5.xlarge to build the index followed by c5.12xlarge to do the mapping. Mapping took a little under an hour per sample.

## What this code does
The notebook extracts the data from the alevin binary format (in R), separates-out the "unspliced" and "spliced" counts using splitSE (from tximeta), and then converts the data into a format to be used by scVelo (anndata) in Python. I then remove droplets with very low and very high counts and select only genes with high dispersion across all the samples which will be used for velocity estimation.

From here, various processing calculations are performed including normalization. Data is stored along the way. After scvelo.recover\_dynamics() is run the anndata object contains large amounts of data (due to dense matrices). Data is written to disk in a format which can be quickly loaded and plotted to avoid recomputation.

Plots are then generated. I still need to tweak the velocity fitting parameters as they are a bit messy but they give the gist of the dynamics.

## Conclusions
It is clear so far from the analysis that there is contamination of the CD4 T-cells with non-T cells in donor 1 (but not donor 2). My guess is that these are likely the feeder cells (H80) which are used to keep the immune cells activated and alive. This population can be clearly seen by examining CD3 expression (a T-cell marker) in figures:  scvelo\_d1i\_CD3d\_umap.png and scvelo\_d1u\_CD3d\_umap.png

Second, it is clear that there is a population of cells that lack beta-2-microglobulin (B2M) expression but express CD3. This subpopulation can be clearly seen in figures scvelo\_d1u\_B2M\_umap.png, scvelo\_d2u\_B2M\_umap.png, scvelo\_B2M\_CD3d\_umap.png. If this turns out to be real it could suggest that a subset of Tcells can downregulate B2M - suggesting a possible mechanism by which T-cells can survive for long periods of time by avoiding CTLs (which require expression of B2M as part of MHC-1 in order to mediate killing). I am looking into what other genes correlate with B2M expression in this dataset in hopes of a hint for what might be causing loss of B2M.

## Future work
1. Mitochodrial gene expression.
2. Identify genes that correlate with B2M.
3. Adjust RNA velocity fitting params to clean-up plots.

## Acknowledgements
T Bradley, G Ferrari, BF Haynes, DM Margolis, EP Browne. Single-cell analysis of quiescent HIV infection reveals host transcriptional profiles that regulate proviral latency. Cell reports 25 (1), 107-117. https://doi.org/10.1016/j.celrep.2018.09.020.

[This](https://combine-lab.github.io/alevin-tutorial/2020/alevin-velocity/) tutorial on using alevin with RNA velocity.
