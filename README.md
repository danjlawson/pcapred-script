# *pcapred* script format

This R code allows you to predict PCA values in the UK Biobank reference data as a **script**. You do not need to be an R user to access it, as long as you have R installed. It uses the R package [pcapred](https://github.com/danjlawson/pcapred) which includes key reference data derived from the UK Biobank.

## Getting started

Get started with:

```{sh}
Rscript pcapred.R -h
```

It is likely that you do not have all of the libraries required to run the script. If not, the script will attempt to install them. If this fails, you will have to install them manually. Please contact the author if this occurs.

### What you need:

Your data, in plink bim/bed/fam format, pruned to the set of SNPs that are useful for PCA. For pruning, we recommend that you use plink. For example, this is how we pruned the 1000 Genomes data to the required SNP set:

```{sh}
plink2 --bfile data/1000G_phase3_common_norel --extract ../flashpca/pca_input_files/pca_input_final.bim --make-bed --out data/1000G_forpca
```

(Technical details: this needs to contain significant overlap in SNPs with the reference. You may need to impute to 1000 Genomes in order to use the default UK Biobank reference dataset, though you can build alternative references. You also have to ensure that your data are on the correct "Genome Build" (37) and strand - forward. However, you don't have to worry about coding of reference/alternative alleles.)

### Example application

A small subset of 1000G data is included as an example in `examples/1000G_smallsubset`. To project this into the first 18 PCs of UK Biobank, simply do:

```{sh}
./pcapred.R -f examples/1000G_smallsubset
```

This creates the file `examples/1000G_smallsubset.eigenvec.pred` which contains the projected PCs1-18 from the UK Biobank, in plink's --covar format.

### Further reference data

We have provided reference datasets to access the first 18 PCs (included in [pcapred](https://github.com/danjlawson/pcapred) and [100/200 PCs](https://github.com/danjlawson/pcapred.largedata) from the UK Biobank variation. (They are in the /inst/extdata/ directory. Due to file size constraints, the 200 PCs version is created  the first time the package is called from R.)

```{sh}
./pcapred.R -f examples/1000G_smallsubset --ref /path/to/ukb_pcs_100
```

There are 3 files:
- .afreq.gz # Contains the allele frequency information as output by plink2 --freq (then gzipped)
- .val # Contains the PC eigenvalues as output by flashpca_x86-64 --outval
- .load.gz # Contains the SNP weights from the PC eigenvalues as output by flashpca_x86-64 --outload (then gzipped)

From these, we can predict the PCs of any external data file that has a large number of overlapping SNPs with the reference, and perform essesntial QC.

### Your data

You need to get your data into plink2 bim/bed/fam format.

For parallelisation and memory usage, you might want to split the input data into groups of say 1000. 1000 takes about 70 minutes to run.

You can then give the data to pcapred with:

```
./pcapred.R --file dataroot
```

If your data are large you will have problems.  In this case you probably are best doing tyour own QC and using flashpca directly.

## Output

By default pcapred will put the output in <input>.eigenvec.pred. You can specify another location with:

```
pcapred.R --out outputfile
```

### Installing required packages

The script should attempt to install the required packages for you. If this fails, you need to install the following R packages:

```
install.packages("optparse")
install.packages("data.table")
install.packages("R.utils")
remotes::install_github("danjlawson/pcapred")
```

### Testing

If you have run flashpca on these samples, copy/link the pcs output into dataroot.pcs and use

```
pcapred.R --test
```

### Executing the script

Either:
- Copy pcapred.R to your path, and do "chmod +x pcapred.R", or
- call it with Rscript /path/to/pcapred.R.

### Getting help

You can get a list of options with

```
pcapred.R --help
```

## Usage notes

At the moment:

- The script can handle SNPs that are not in the reference. They get removed, and the remaining SNPs are reweighted to retain the same mean and variance of the predicted distribution.
- The script can handle SNPs that are missing in your data, either missing the whole SNP or missing for a specific individual. They get removed on a case-by-case basis.
- The script tries to standardise the reference and alternative alleles to match. It only uses the REFERENCE allele for this, so error catching is incomplete.
- The script may not cope if your data are on the WRONG GENOME BUILD and STRAND. UK Biobank is on forward strand of B37.
- The predictions will degrade if many SNPS are missing. You are encouraged to run imputation before passing the data to it!

# Working with your own datasets

## Making a test dataset

This is how I generated the 1000 Genomes example dataset dataset. In R:

```{r}
remote::install_github("privefl/bigsnpr")
library("bigsnpr")
download_1000G("data")
system("sort -R data/1000G_phase3_common_norel.fam | head -n 100 > data/1000G_phase3_common_norel.indstokeep")
system("plink2 --bfile data/1000G_phase3_common_norel --keep data/1000G_phase3_common_norel.indstokeep --extract ../flashpca/pca_input_files/pca_input_final.bim --make-bed --out data/1000G_smallsubset")
```

## Making your own data from UK Biobank

This is how I generated the 200 PCs data for the UK Biobank dataset.

```{sh}
plink2 --bfile pca_input_files/pca_input_final --freq --out pca_input_files/pca_input_final_200
gzip pca_input_files/pca_input_final_200.afreq pca_input_files/pca_input_final_200.raw
gzip flashpca_200.load
```

## Final thoughts

There is more to be done:

* Checking of data with different coding
* Automatic imputation.
* Creation of additional SNP sets.
* Automatic choice of SNP sets.

## Licence Information

Author: Daniel Lawson (dan.lawson@bristol.ac.uk)
Date: 2020
Licence: This code is released under the GPLv3.
