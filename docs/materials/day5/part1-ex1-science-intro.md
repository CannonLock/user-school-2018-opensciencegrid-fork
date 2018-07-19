---
status: done
---

<style type="text/css"> pre em { font-style: normal; background-color: yellow; } pre strong { font-style: normal; font-weight: bold; color: \#008; } </style>

Friday Exercise 1.1: Learn about Joe's Desired Computing Work
=============================================================

In today's set of hands-on exercises, you'll be examining a set of pre-existing job submissions (scripts and submit files) and then coordinating them to run as a single workflow using a DAG. Along the way, you'll need to test some of the already written submit files. The first part of this exercise introduces the overall steps for the workflow and what pieces already exist (that you won't need to write yourself).

Joe's Science and Workflow
--------------------------

Joe Biologist studies the influence of genetic differences (genotypes) on the growth traits (phenotypes) of a common crop. He has come to you with some computing work he’d like to automate as a workflow, so that he can scale out his research and run it on a much larger HTC compute system. 

Joe starts with a spreadsheet containing phenotype measurements and genotype information for hundreds of plants.  Each phenotype measurement corresponds to a trait that he's interested in learning about.  In order to understand the relationship of these trait phenotypes to the underlying genetic material (the genotypes), he first generates thousands of combinations of the trait's phenotype with all the genotypes.  These **permutations** represent hypothetical phenotype-genotype connections that could be meaningful for that trait.  Each of these permutations is then run through a **QTL mapping** process, which sifts through all the permutations to find the genotypes that are most important for the trait in question.  Joe has to repeat this process (generating genotype-phenotype permutations and then running a QTL mapping on those permutations) separately for each trait he's studying.  

Joe's Current Setup
-------------------

Joe has run some of this work on a high throughput system -- submitting individual jobs on a small, dedicated HTCondor cluster where he doesn’t have to request CPU, disk, or RAM.  

(As you learn more details below, it might be helpful to draw a diagram that describes the steps of his workflow.)

Specifically, for one trait, his workflow looks like this: 

1. Uploading an input file (`input.csv`) to the submit server that has *all* of the genotypic and phenotypic information for multiple traits.  
1. Joe then uses a submit file, perl wrapper script (`runR.pl`) and custom R script (`run_perm.R`) to submit jobs that generate the permutations for a single trait.  The perl script is the job's executable.  Besides setting up the job environment, it takes several arguments: 
	- The name of the R script that creates the permutations
	- Which trait (and therefore which phenotype) it's using to create permutations
	- How many permutations to create per job
	- When running multiple permutation jobs at once for a trait, a number to identify the job. 
1. After the permutation job(s), a shell script creates a tarball of the permutation files for that trait (which may have been generated by multiple jobs).  This is a short-running script that runs directly on the submit server.  
1. The second job (the QTL mapping step) uses the tarball of trait permutations from the previous step, the same perl wrapper script (`runR.pl`), and a different R script (`run_qtl.R`) to run the QTL mapping on the permutations generated in the first batch of jobs.  
1. Finally, a second shell script creates a tarball of the QTL mapping output on the submit server.   

He has to perform all of these steps for each trait he wants to analyze.  

That's a lot of steps!  Joe Biologist is getting annoyed by how many manual job submissions and summary scripts he has to run, especially as he's thinking of scaling out his work in a way that would be much more cumbersome.  As one example, when Joe has previously run the **permutation** step (generating 10,000 permutations for a single trait) as one Condor job on his own cluster, the job runs for hours.  To solve that problem in the past, he split the single permutation-creating Condor job into multiple, shorter Condor jobs, each creating fewer permutations, but he doesn’t remember the details of how many jobs and how many permutations per job worked well. Furthermore, Joe would like to scale up to 100,000 permutations (from 10,000) per trait this time.   Unlike the **permutation** step, the **QTL** step finishes quite quickly; even with 10,000 permutations completed for each trait, the **QTL** step completes within several minutes.

Joe knows that he needs to modify and optimize his submissions to run jobs that create more permutations and that DAGMan might help him automate all these steps.  He's never used DAGMan, so he's asking you to help him organize all the steps into an optimized DAG workflow.

View Joe's files
----------------

**(but don't do anything with them until the next [exercise](part1-ex2-plan-workflow.md), where you'll plan necessary developments of a workflow for Joe.)**

Log in to `learn.chtc.wisc.edu` and move to a desired location in your home directory. Enter the following commands to download and decompress/untar Joe’s job ingredients:

```console
username@learn $ wget http://proxy.chtc.wisc.edu/SQUID/osgschool18/WorkflowExercise.tar.gz
username@learn $ tar -xzf WorkflowExercise.tar.gz
```

You can now navigate into the `WorkflowExercise` directory to view the full ingredients for Joe’s Computing work. Review all of these files based upon the information below and refer back to it as you proceed through the remaining exercises (1.2, 2.1, 2.2). (If you've been drawing a diagram of Joe's work so far, feel free to annotate with some of this information.)

Based upon Joe’s description and submit files you determine the following details for analyzing **trait 1**: 

### Permutation Step

The command the job should run: `./runR.pl 1_$(Process) run_perm.R 1 $(Process) 10000`

The submit file for the permutation jobs is `permutation1.submit`, including the following: 

```file
executable = runR.pl
arguments = 1_$(Process) run_perm.R 1 $(Process) 10000
transfer_input_files = run_perm.R, input.csv, RLIBS.tar.gz
```

The arguments mean the following:

-   `1_$(Process)`, used by `runR.pl` to name its log file
-   `run_perm.R`, which R script we want to run
-   `1`, trait column in `input.csv`
-   `$(Process)`, used by `run_perm.R` for naming the permutation output
-   `10000`, indicates that 10,000 permutations should be done for each individual job process.  If we use this submit file to create multiple jobs, each of them will create 10,000 permutations.  

As output, this job will create:

-   a series of .log and .out files, named from the first argument (`1_$(Process)`). This looks like: `runR.1_0.out`, `runR.1_0.log`, `runR.1_0.err`
-   the actual output file, named using the `1` and `$(Process)` arguments, like so: `perm_part.1_0.Rdat`

### Combine Permutation Output

`tarit.sh` is run after the trait's **permutation** step with the column number as an argument to compress `perm_part.1_0.Rdat` (or potentially multiple such files named according to `perm_part.1_*.Rdat`, see above) for the QTL step.

Sample execution: 
```console
username@host $ ./tarit.sh 1
```
where `1` is the trait column number.

### QTL Mapping Step

The command the job should run: `./runR.pl qtl_1 qtl.R 1`

The submit file for the permutation jobs is `qtl1.submit`, containing the following: 

```file
executable = runR.pl
arguments qtl_1 qtl.R 1
transfer_input_files = qtl.R, input.csv, RLIBS.tar.gz, perm_combined_1.tar.gz
```

Note that `perm_combined_1.tar.gz` was created by `tarit.sh`.

The arguments mean the following:

-   `qtl_1`: used by `runR.pl` to name its log / output / error files
-   `qtl.R`: which R script we want to run
-   `1`: trait column in `input.csv`, will also be used to name the output files

As output, this job will create:

-   a series of .log and .out files, named from the first argument (`qtl_1`).
-   several output files named for the trait argument `1`: `perm_combined_1.Rdat`, `perm_summary_1.txt`, `sim_geno_results_1.Rdat`, `qtl_1.Rdat`, `refined_qtl_summary_1.txt`, `refined qtl_1.Rdat`, and `fit_qtl_results_1.Rdat`

### Combine All Output

`results_1.tar.gz` is made by running `taritall.sh` with the column number as an argument after the QTL step for that trait finishes, where "1" reflects the trait column number in the output above.

Sample execution: 

```console
username@host $ ./taritall.sh 1
```
where `1` is the trait column number.

### Other Files

The other files in the workflow directory (`permutation2.submit`/`qtl2.submit` and `permutation3.submit`/`qtl3.submit`) are used to submit the **permutation** and **qtl** jobs for traits 2 and 3, respectively.  
