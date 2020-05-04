# pcapred

This R code allows you to predict PCA values in the UK Biobank reference data as a **script**. You do not need to be an R user to access it, as long as you have R installed. It uses the R package [pcapred](https://github.com/danjlawson/pcapred) which includes key reference data derived from the UK Biobank.

## Getting started

Get started with:

```{sh}
Rscript pcapred.R -h
```

It is likely that you do not have all of the libraries required to run the script. If not, the script will attempt to install them. If this fails, you will have to install them manually. Please contact the author if this occurs.

### What you need:

Your data, in plink bim/bed/fam format.

(Technical details: this needs to contain significant overlap in SNPs with the reference. You may need to impute to 1000 Genomes in order to use the default UK Biobank reference dataset, though you can build alternative references.)

### Example application

A small subset of 1000G data is included as an example in `examples/1000G_smallsubset`. To project this into the first 18 PCs of UK Biobank, simply do:

```{sh}
./pcapred.R -f examples/1000G_smallsubset
```

This creates the file `examples/1000G_smallsubset.eigenvec.pred` which contains the projected PCs1-18 from the UK Biobank, in plink's --covar format.

### Further reference data

We have provided reference datasets to access the first 100 or 200 PCs from the UK Biobank variation. These are available for download at XXXXX.

```{sh}
pcapred.R --ref /path/to/pca_input_final_200
```

There are 3 files:
- .afreq.gz # Contains the allele frequency information as output by plink2 --freq (then gzipped)
- .val # Contains the PC eigenvalues as output by flashpca_x86-64 --outval
- .load.gz # Contains the SNP weights from the PC eigenvalues as output by flashpca_x86-64 --outload (then gzipped)

From these, we can predict the PCs of any external data file that has a large number of overlapping SNPs with the reference, and perform essesntial QC

### Your data

You need to get your data into plink2 bim/bed/fam format.

For parallelisation and memory usage, you might want to split the input data into groups of say 1000. 1000 takes about 70 minutes to run.

You can then give the data to pcapred with:

```
./pcapred.R --file dataroot
```

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
library*("bigsnpr")
download_1000G("data")
system("sort -R data/1000G_phase3_common_norel.fam | head -n 100 > data/1000G_phase3_common_norel.indstokeep")
system("plink2 --bfile data/1000G_phase3_common_norel --keep data/1000G_phase3_common_norel.indstokeep --extract ../flashpca/pca_input_files/pca_input_final.bim --make-bed --out data/1000G_smallsubset")
```

## Making your own data from UK Biobank

```{sh}
plink2 --bfile pca_input_files/pca_input_final --freq --out pca_input_files/pca_input_final_200
gzip pca_input_files/pca_input_final_200.afreq pca_input_files/pca_input_final_200.raw
gzip flashpca_200.load
```

## Toy data

This is how I generated a small dataset for testing from UK Biobank.  In bash:

```{sh}
head -n 150 pca_input_files/pca_input_final.fam > pca_input_files/test_keep.fam
head -n 400 pca_input_files/pca_input_final.bim | cut -f2 > pca_input_files/test_extract.bim
plink1.9 --bfile pca_input_files/pca_input_final --keep pca_input_files/test_keep.fam --extract pca_input_files/test_extract.bim --make-bed --out pca_input_files/test
```

## Make summary data
```{sh}
plink2 --bfile pca_input_files/test --freq --out pca_input_files/test
gzip pca_input_files/test.afreq
```

## Perform miss pca
```{sh}
plink2 --bfile pca_input_files/test --pca var-wts --out test
gzip test.eigenvec.var
flashpca_x86-64 --bfile pca_input_files/test --outpc test.pcs -d 50 --outval test.val --outload test.load --outpve test.pve
gzip test.load
zcat test.load.gz | ~/bin/transpose -t --limit 428759x52 | gzip -c > test.tload.gz
```

## Make nomiss data
```{sh}
plink1.9 --bfile pca_input_files/test -geno 0 --out pca_input_files/test_nomiss --make-bed
plink1.9 --bfile pca_input_files/test_nomiss --recode A --out pca_input_files/test_nomiss
plink2 --bfile pca_input_files/test_nomiss --freq --out pca_input_files/test_nomiss
gzip pca_input_files/test_nomiss.afreq
```

## Perform nomiss pca
```{sh}
plink2 --bfile pca_input_files/test_nomiss --pca var-wts --out test_nomiss
gzip test_nomiss.eigenvec.var
flashpca_x86-64 --bfile pca_input_files/test_nomiss --outpc test_nomiss.pcs -d 50 --outval test_nomiss.val --outload test_nomiss.load --outpve test_nomiss.pve
gzip test_nomiss.load
zcat test_nomiss.load.gz | ~/bin/transpose -t --limit 428759x52 | gzip -c > test_nomiss.tload.gz
```

## Final thoughts

There is more to be done:

* Process in batches
* Checking of data with different coding
* Automatic imputation.
* Creation of additional SNP sets.
* Automatic choice of SNP sets.

## Licence Information

Author: Daniel Lawson (dan.lawson@bristol.ac.uk)
Date: 2020
Licence: This code is released under the GPLv3.
