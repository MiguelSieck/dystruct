# Dystruct (Dynamic structure)

1. [Introduction](#introduction)
2. [Installation](#installation)
  * [Quick Start](#quick-start)
3. [Running Dystruct](#running-dystruct)
  * [Input Files](#input-files)
  * [Comparing Runs (Choosing K)](#model-choice)
  * [Running Time](#running-time)
  * [Convergence](#convergence)

## Introduction
Dystruct (Dynamic structure) is a model-based approach for inferring shared ancestry in individuals sampled over time. The input the model is a genotype matrix, sample times for each individual, and the number of putative ancestral populations (K). The output are K-dimensional ancestry vectors denoting the proportion of the the genome of each individual inherited from population K.

The main benefit of Dystruct, when compared to static ancestry approaches such as ADMIXTURE, is Dystruct's emphasis on explaining later populations as mixtures of earlier, pre-existing, populations. That is, later samples tend to appear as mixtures of earlier samples. Because we model samples over time, ancestry components give a direct interpretation of the relationship between populations in the past, and populations today.

## Installation

Dystruct requires a C++ compiler (preferably gcc v6.1 or later).

### macOS and OpenMP
Dystruct depends on the OpenMP API for parallel computation. 
OpenMP is not supported on some compliers, most notably the clang compiler shipped with macOS. Mac users will need to compile Dystruct using a different C++ compiler. To install gcc using homebrew, simply call

```
brew install gcc6
```

If your compiler does not support OpenMP, you will likely see the error

```
clang: error: unsupported option '-fopenmp'
```

You can either remove this flag from the makefile, which disables parallel computation, or install a different compiler per the above directions.

### Quick Start

To compile and run on example data, type:

```
git clone https://github.com/tyjo/dystruct
cd dystruct
make
./bin/dystruct
./run.sh
```


## Running Dystruct

### Input Files
Dystruct requires two files to run: a genotype matrix in the EIGENSTRAT .geno format, and a list (one per line) of generation times for each sample. Example .geno and generation time files are available in `supp/example_data/`. The script `run.sh` runs Dystruct on the example data.

The genotype matrix is a SNPs by individual matrix where each entry encodes the number of non-reference alleles (i.e. 0, 1, 2) for that genotype with no spaces between entries. Missing data is denoted by 9. For example, a file with 10 individuals sampled at 3 sites may look like

```
0121100292
1129991221
0901120000
```

The [convertf](https://github.com/DReichLab/AdmixTools/tree/master/convertf) program converts between several standard formats including: EIGENSTRAT (used by Dystruct), PED, and ANCESTRYMAP.

The generation times file contains one line per individual giving the generation time the individual was alive. Generation times are necessarily imprecise due to uncertainty in carbon-date estimates or estimates of the date for each culture. In practice we found that precise dates are necessary to infer historical relationships.

It is recommended to reduce the number of distinct generation times used to be 15 or fewer, either grouping individuals within the same culture, or grouping generation times into a smaller number of bins.  A script to bin generation times is available under ``supp/scripts/bin_sample_times.py``.


### Arguments
Dystruct takes several command line arguments:

```
Usage:   dystruct [options]

Required Arguments:
	-h, --help                  Print this help message.
	--input FILE                Path to genotype matrix: a LOCI x INDIVIDUAL matrix of genotypes in the
                                    EIGENSTRAT genotype format. Each genotype is denoted by either 0, 1, 2, or 9,
                                    where 9 identifies missing entries. There are no spaces between entries.
                                    See https://github.com/DReichLab/EIG/tree/master/CONVERTF for more details
                                    and converting between standard formats.
	--generation-times FILE     Path to generation times corresponding to the input file. A list of generation
                                    times (one per line) for each sample. Samples are assumed to be in the same
                                    order as the columns of the input matrix.
	--output STR                A prefix for output files.
	--npops INT                 Number of populations.
	--nloci INT                 Number of loci. This should match the number of loci in the input file.
	--pop-size INT              Effective population size for all populations.
	--seed INT                  Random seed used to initialize variational parameters

Optional Arguments:
	--hold-out-fraction DOUBLE  (=0) Optional. Partitions nloci * hold_out_fraction loci into a hold out
                                    set. The hold out set contains at most one individual per site.
	--hold-out-seed INT         (=28149) Optional. Random seed used to partition SNP data into hold out
                                    and training sets. Use the same seed across replicates to fix the hold
                                    out set.
	--epochs INT                (=50) Optional. Number of epochs to run before terminating.
	--labels FILE               Optional. Experimental. Population label file path for supervised analysis.
                                    Labels should be in {0,...,npops - 1}. One label per line in the same order
                                    as the input matrix. Individuals without a population assignment should be
                                    labeled by -1.
```


### Parallel Computation
Dystruct parallelizes certain sections of its algorithm for faster computation using the OpenMP library. To specify the number of threads OpenMP looks for the environment variable OMP\_NUM\_THREADS. You can control the number of threads the software uses by setting this environment variable. For example, if you want to use 2 threads call:

```
export OMP_NUM_THREADS=2
./bin/dystruct --input FILE \
               --generation-times FILE \
               --output STR \
               --nloci INT \
               --pop-size INT \
               --seed INT
```

For best performance set the number of threads to the number of ancestral populations (K):

```
export OMP_NUM_THREADS=K
```


### Output Files
Dystruct outputs two files with point estimates for inferred parameters, and a temporary file to monitor convergence. The prefix of these files is specified through the --output command line argument. The inferred parameter files are:

- freqs : the inferred allele frequencies at all time steps
- theta : the inferred ancestry proportions for all samples

Dystruct also outputs a temporary file, temp\_theta, with the current estimates of the variational parameters for ancestry proportions. Each row is an individual, and each column a population. The number in each entry refers to the number of loci currently assigned to a population.


### Model Choice (Choosing K)
Dystruct takes two optional arguments to hold out a subset of loci to model evaluation. These are `--hold-out-fraction` and `--hold-out-seed`. `--hold-out-fraction` is a number in [0,1] that gives a proportion of sites to partition into a hold out set. At most one site per individual is held out. Thus, if `--hold-out-fraction` is equal to 1, one locus in one individual is put into the hold out set for each SNP. Keeping the `--hold-out-seed` consistent across runs ensures that the same set of loci is held out for each run.

After convergence, Dystruct outputs the conditional log likelihood on the hold out set. This is the binomial log likelihood given the current point estimates of ancestry proportions and allele frequencies. The final value can --- and should --- be used to compare runs on the same K, where the run with the highest conditional log likelihood is chosen. Similarly, the conditional log likelihood can be used to choose the "best" value of K. Nonetheless, we emphasize that results should be intepreted across multiple K. 


### Running Time
Running time per iteration primarly depends on three factors. In order of importance, these are: i) how many times the algorithm has previously seen a loci, ii) number of time points, and iii) number of individuals. Earlier iterations tend to be much slower because it takes longer for the local parameter estimates at each locus to converge. Thus, performance during earlier iterations should not be used to estimate run time. In our experience, setting `OMP_NUM_THREADS=K`, Dystruct converged in less than 24 hours on a dataset of ~1800 individuals at ~300000 loci across 11 time points.


### Convergence
By default Dystruct terminates after 50 epochs, where one epoch occurs every `NLOCI` iterations. Using simulations we determined that this was adequate for convergence, but users with smaller datasets (<50000 loci) may consider increasing this number to 100. This setting can be changed using the `--epochs` argument.

