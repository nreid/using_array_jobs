# Job Arrays on Xanadu

A guide to splitting up big jobs using slurm job arrays on UConn's Xanadu cluster. 

See the scripts directory of the repository for examples. 

## What is a job array?

According to the [SLURM](https://slurm.schedmd.com/job_array.html) documentation:

>Job arrays offer a mechanism for submitting and managing collections of similar jobs quickly and easily

That is, if you want to do something very similar many times, you can split the job up into an array of smaller tasks to parallelize it. 

Some examples of situations where array jobs can be helpful:

- Aligning many fastq files to a reference genome.
- Evaluating many positions in the genome (e.g. variant calling). 
- Simulating and evaluating many datasets. 

An example of a trivial array job script, which would be submitted using `sbatch` is below. To run this as a test, create a directory `mkdir array_test`, save the script there using a text editor such as `nano` and submit it to the job scheduler using the command `sbatch`. 


```bash
#!/bin/bash
#SBATCH --job-name=MY_JOB_ARRAY_NAME
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -c 1
#SBATCH --mem=1G
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --array=[1-1000]%20
##SBATCH --mail-type=ALL
##SBATCH --mail-user=YOUR.EMAIL@uconn.edu
#SBATCH -o %x_%A_%a.out
#SBATCH -e %x_%A_%a.err


echo "host name : " `hostname`
echo This is array task number $SLURM_ARRAY_TASK_ID


```

The job will produce 2 files for each task. Each `.out` file contains the hostname and the task number for each task, and the `.err` file contains any corresponding text printed to the standard error stream. 

For each task, the variable SLURM_ARRAY_TASK_ID is set to the task number. These numbers can range from 0 to 1000 (currently the maximum array size on Xanadu), but can be any set of numbers or ranges, separated by commas. The numbers are specified on this line:

```bash
#SBATCH --array=[1-1000]%20
```

The bracketed range gives the task numbers, and `%20` indicates that 20 tasks should run simultaneously. The other SLURM parameters should be set to match the requirements of each task. For example, if you are aligning a fastq file using `bwa` and you want to use 4 CPUs, you would set `#SBATCH -c 4`. 

The key to making these job arrays useful is in how you use the SLURM_ARRAY_TASK_ID variable. 

## Iterating over a list of files

As an example, let's pretend we have Illumina sequencing data from 25 samples. For each sample we have paired sequences in separate files, each named Sample_A.R1.fastq, Sample_A.R2.fastq, etc. We can make dummy files for this exercise:

```bash
# make a new directory to hold test files and cd into it
mkdir array_test_2
cd array_test_2

touch Sample_{A..Y}.R1.fastq
touch Sample_{A..Y}.R2.fastq
```

Say we wanted to use an array job to align each of these samples in parallel. One way to approach this is to create an array variable containing the list of files to analyze, and then use the SLURM_ARRAY_TASK_ID to retrieve elements of the list. 

An array job script that does this might look like this:

```bash
#!/bin/bash
#SBATCH --job-name=MY_JOB_ARRAY_NAME
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -c 1
#SBATCH --mem=1G
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --array=[0-24]%20
##SBATCH --mail-type=ALL
##SBATCH --mail-user=YOUR.EMAIL@uconn.edu
#SBATCH -o %x_%A_%a.out
#SBATCH -e %x_%A_%a.err


echo "host name : " `hostname`
echo This is array task number $SLURM_ARRAY_TASK_ID

# create an array variable containing the file names
FILES=($(ls -1 *.R1.fastq))

# get specific file name, assign it to FQ1
	# note that FILES variable is 0-indexed so
	# for convenience we also began the task IDs with 0
FQ1=${FILES[$SLURM_ARRAY_TASK_ID]}
# edit the file name to refer to the mate pair file and assign that name to FQ2
FQ2=$(echo $FQ1 | sed 's/R1/R2/')
# create an output file name
OUT=$(echo $FQ1 | sed 's/.R1.fastq/.sam/')
# write the input filenames to the standard output to check that everything ran according to expectations. 
echo $FQ1 $FQ2

# we won't actually try to align these fake files here but it might look like:

# module load bwa
# bwa mem refgenome.fa $FQ1 $FQ2 > $OUT

echo Files $FQ1 and $FQ2 were aligned by task number $SLURM_ARRAY_TASK_ID on $(date)

```

We created the array variable by `FILES=($(ls -1 *.R1.fastq))`, retrieving all files ending in `*.R1.fastq`. 

We then retrieve each individual `FQ1` by using `SLURM_ARRAY_TASK_ID` to grab one element of `FILES`. We then get the paired ends by modifying `FQ1` using `sed`: `FQ2=$(echo $FQ1 | sed 's/R1/R2/')` and defining the output file, `OUT` similarly. 

Another approach to make this slightly more robust would be to generate the FILES array using `find`:

`FILES=($(find /path/to/input_files/ -name "*R1.fastq"))`

This would use the full path for each file and allow the script to be run from any directory. 

If you had a small number of files, you could also avoid using the search pattern altogether and define the array variable inside the script by writing out the file names: `FILES=(Sample_A.R1.fastq, Sample_B.R1.fastq. Sample_C.R1.fastq)`

The approach outlined here would work for any situation where you need to iterate an analysis over many files. 

## Iterating over varying input parameters

This approach of defining the list of items to be operated on inside the array job script works fine for lists of files (as long as you define matching patterns, file names, and directories that won't collide). But when the variables you want to iterate over are not straightforward manipulations of the task ID or file names, such as targeting many genomic regions, you need another approach. 

In those cases you may want to use a file that contains all the relevant information, and extract what you need for each task. 

As an example, let's say we want to call variants on human whole genome sequencing data. Instead of letting one job churn through the whole thing sequentially, we might break the job into 10 megabase chunks and run 20 jobs simultaneously to speed it up. 

We can first define these 10mb windows using `bedtools`. 

```bash
# make a new directory to hold test files and cd into it. 
mkdir array_test_3
cd array_test_3

# load bedtools
module load bedtools

# this is a tab delimited file giving chromosome names and lengths for the human genome
	# you can make your own version of this file with a reference genome and "samtools faidx"
GEN=/isg/shared/databases/alignerIndex/animal/hg38_ucsc/hg38_STAR/chrNameLength.txt

bedtools makewindows -g $GEN -w 10000000 >10mb.win.bed
```

The resulting bed file, `10mb.win.bed` has 791 lines and looks like this:

```bash
chr1	0	10000000
chr1	10000000	20000000
chr1	20000000	30000000
chr1	30000000	40000000
chr1	40000000	50000000
chr1	50000000	60000000
```

We want to run the variant caller over each region as a separate task. We could specify the job array this way:

```bash
#!/bin/bash
#SBATCH --job-name=MY_JOB_ARRAY_NAME
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -c 1
#SBATCH --mem=1G
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --array=[1-791]%20
##SBATCH --mail-type=ALL
##SBATCH --mail-user=YOUR.EMAIL@uconn.edu
#SBATCH -o %x_%A_%a.out
#SBATCH -e %x_%A_%a.err


echo "host name : " `hostname`
echo This is array task number $SLURM_ARRAY_TASK_ID

# put relevant data in variables
CHR=$(sed -n ${SLURM_ARRAY_TASK_ID}p 10mb.win.bed | cut -f 1)
START=$(expr $(sed -n ${SLURM_ARRAY_TASK_ID}p 10mb.win.bed | cut -f 2) + 1)
STOP=$(sed -n ${SLURM_ARRAY_TASK_ID}p 10mb.win.bed | cut -f 3)

# define region variable
	# for most tools in the format chr:start-stop
REGION=${CHR}:${START}-${STOP}

echo This task will analyze region $REGION of the human genome. 

# subsequent lines would define input and output file names and the variant caller, e.g:

# module load freebayes
# freebayes -r $REGION -f ref.fa aln.bam >var.${REGION}.vcf

```

We specify the size of the job array as `#SBATCH --array=[1-791]%20`. Note that it begins at 1 in this case. We use `sed` to print only the line of the file corresponding to the array task number and pipe that to `cut` to pull out the column that corresponds to the bit of data we want, placing that in a shell variable. Note that for `START`, we add 1 to the number in the bed file because a quirk of the BED format is that the start column is 0-indexed while the end column is 1-indexed. The region specification format of sequence:start-stop usually expects the start and stop positions to both be 1-indexed. 

This would result in 791 jobs being run 20 at a time and produce as many output vcf files. The final step would probably be to use a package like `bcftools` or `vcflib` to filter and combine these into a single vcf. 

This approach could be generalized to any situation where you have a repetitive task you wish to parallelize, but input parameters that vary for each task. 

