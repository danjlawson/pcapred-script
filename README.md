# pcapred

This R code allows you to predict PCA values in the UK Biobank reference data as a **script**. You do not need to be an R user to access it, as long as you have R installed. It uses the R package [pcapred](https://github.com/danjlawson/pcapred) which includes key reference data derived from the UK Biobank.

## Getting started

Get started with:

```
Rscript pcapred.R -h
```

It is likely that you do not have all of the libraries required to run the script. If not, the script will attempt to install them. If this fails, you will have to install them manually but 

### Reference data

You need the reference data. This is on Bluecrystal phase 3 at:

/projects/MRC-IEU/scratch/pcapred/pca_input_files/pca_input_final_200*

and can be obtained by emailing Daniel Lawson (dan.lawson@bristol.ac.uk) if you don't have access to this location. It is not sensitive. You need to **unzip** these files, and you will specify them with:

```
pcapred.R --ref /path/to/pca_input_final_200
```

There are 3 files:
- .afreq.gz # Contains the allele frequency information as output by plink2 --freq (then gzipped)
- .val # Contains the PC eigenvalues as output by flashpca_x86-64 --outval
- .load.gz # Contains the SNP weights from the PC eigenvalues as output by flashpca_x86-64 --outload (then gzipped)

From these, we can predict the PCs of any external data file that has a large number of overlapping SNPs with the reference.

### Your data

You need to get your data into plink2 bim/bed/fam format.

For parallelisation and memory usage, you might want to split the input data into groups of say 1000. 1000 takes about 70 minutes to run.

You can then give the data to pcapred with:

```
./pcapred.R --file dataroot
```

## Output

By default pcapred will put the output in input.eigenvec.pred. You can specify another location with:

```
pcapred.R --out outputfile
```

### Installing required packages

You need to install the following R packages:

```
install.packages("optparse")
install.packages("data.table")
install.packages("R.utils")
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

- The script can handle SNPs that are not in the reference. They get removed.
- The script can handle SNPs that are missing in your data, either missing the whole SNP or missing for a specific individual. They get removed on a case-by-case basis.
- The script tries to standardise the reference and alternative alleles to match. It only uses the REFERENCE allele for this, so error catching is incomplete.
- The script may not cope if your data are on the WRONG GENOME BUILD and STRAND. UK Biobank is on forward strand of B37 (I think!)
- The predictions will degrade if many SNPS are missing. You are encouraged to run imputation before passing the data to it!

## Examples

See:

```
Rscript pcapredtest.R
```

## Making a test dataset

This is how I generated the test dataset:

```
#############
## TOY DATA
head -n 150 pca_input_files/pca_input_final.fam > pca_input_files/test_keep.fam
head -n 400 pca_input_files/pca_input_final.bim | cut -f2 > pca_input_files/test_extract.bim
plink1.9 --bfile pca_input_files/pca_input_final --keep pca_input_files/test_keep.fam --extract pca_input_files/test_extract.bim --make-bed --out pca_input_files/test

## Make summary data
plink2 --bfile pca_input_files/test --freq --out pca_input_files/test
gzip pca_input_files/test.afreq

## Perform miss pca
plink2 --bfile pca_input_files/test --pca var-wts --out test
cat test.eigenvec.var | ~/bin/transpose -t --limit 428759x52 | gzip -c > test.eigenvec.tvar.gz
gzip test.eigenvec.var
flashpca_x86-64 --bfile pca_input_files/test --outpc test.pcs -d 50 --outval test.val --outload test.load --outpve test.pve
gzip test.load
zcat test.load.gz | ~/bin/transpose -t --limit 428759x52 | gzip -c > test.tload.gz

## Make nomiss data
plink1.9 --bfile pca_input_files/test -geno 0 --out pca_input_files/test_nomiss --make-bed
plink1.9 --bfile pca_input_files/test_nomiss --recode A --out pca_input_files/test_nomiss
plink2 --bfile pca_input_files/test_nomiss --freq --out pca_input_files/test_nomiss
gzip pca_input_files/test_nomiss.afreq

## Perform nomiss pca
plink2 --bfile pca_input_files/test_nomiss --pca var-wts --out test_nomiss
cat test_nomiss.eigenvec.var | ~/bin/transpose -t --limit 428759x52 | gzip -c > test_nomiss.eigenvec.tvar.gz
gzip test_nomiss.eigenvec.var
flashpca_x86-64 --bfile pca_input_files/test_nomiss --outpc test_nomiss.pcs -d 50 --outval test_nomiss.val --outload test_nomiss.load --outpve test_nomiss.pve
gzip test_nomiss.load
zcat test_nomiss.load.gz | ~/bin/transpose -t --limit 428759x52 | gzip -c > test_nomiss.tload.gz
```

## Final thoughts

There is more to be done:

* Process in batches
* Checking of data with different coding
* Automatic imputation/checking?

