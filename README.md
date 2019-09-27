# Job Arrays on Xanadu

A guide to splitting up big jobs using slurm job arrays on UConn's Xanadu cluster. 

## What is a job array?

According to the [SLURM](https://slurm.schedmd.com/job_array.html) documentation:

>Job arrays offer a mechanism for submitting and managing collections of similar jobs quickly and easily

That is, if you want to do something very similar many times, you can split the job up into an array of smaller tasks to parallelize it. 

Some examples of situations where array jobs can be helpful:

- Aligning many fastq files to a reference genome.
- Evaluating many positions in the genome (e.g. variant calling). 
- Simulating and evaluating many datasets. 

An example of a trivial array job script is below:


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
echo This is array job number $SLURM_ARRAY_TASK_ID


```

This job simply prints the hostname and the task number for each task in the array. 

For each task, the variable SLURM_ARRAY_TASK_ID is set to the task number. These numbers can range from 0 to one less than the max number of jobs (on Xanadu this is ####), but can be any set of numbers or ranges, separated by commas. The numbers are specified on this line:

```bash
#SBATCH --array=[1-1000]%20
```

The bracketed range gives the task numbers, and `%20` indicates that 20 tasks should run simultaneously. The other SLURM parameters should be set to match the requirements of each task. For example, if you are aligning a fastq file using `bwa` and you want to use 4 CPUs, you would set `#SBATCH -c 4`. 

The key to making these job arrays useful is in how you use the SLURM_ARRAY_TASK_ID variable. As an example, let's make 50 empty files, each named A.R1.fastq, A.R2.fastq ..., as if we had paired end reads from 25 samples. 

```bash
touch {A..Y}.R1.fastq
touch {A..Y}.R2.fastq
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
#SBATCH --array=[0-25]%20
##SBATCH --mail-type=ALL
##SBATCH --mail-user=YOUR.EMAIL@uconn.edu
#SBATCH -o %x_%A_%a.out
#SBATCH -e %x_%A_%a.err


echo "host name : " `hostname`
echo This is array job number $SLURM_ARRAY_TASK_ID

# create an array variable containing the file names
FILES=($(ls -1 *.R1.fastq))

# get specific file name
	# note that FILES variable is 0-indexed so
	# for convenience we also began the task IDs with 0
FQ1=${FILES[$SLURM_ARRAY_TASK_ID]}
FQ2=$(echo $FQ1 | sed 's/R1/R2/')
OUT=$(echo $FQ1 | sed 's/.R1.fastq/.bam/')
echo $FQ1 $FQ2

# we won't actually try to align these fake files here but it might look like:

# module load bwa
# bwa mem refgenome $FQ1 $FQ2 > $OUT

echo Files $FQ1 and $FQ2 were aligned by task number $SLURM_ARRAY_TASK_ID on $(date)

```

We create the array variable by `FILES=($(ls -1 *.R1.fastq))`, retrieving all files ending in `*.R1.fastq`. 

We then retrieve each individual `FQ1` by using `SLURM_ARRAY_TASK_ID` to grab one element of `FILES`. We then get the paired ends by modifying `FQ1` using `sed`: `FQ2=$(echo $FQ1 | sed 's/R1/R2/')` and defining the output file, `OUT` similarly. 

Another approach to make this slightly more robust would be to generate the FILES array using `find`:

`FILES=($(find /path/to/input_files/ -name "*R1.fastq"))`

This would use the full path for each file and allow the script to be run from any directory. 

If you had a small number of files, you could also avoid using the search pattern altogether and define the array variable inside the script by `FILES=(A.R1.fastq, B.R1.fastq. C.R1.fastq)`

The approach outlined here would work for any situation where you need to iterate an analysis over many files. 

___


This approach of defining the list of items to be operated on inside the array job script works fine for lists of files (as long as you define matching patterns, file names, and directories that won't collide). But for targeting many genomic regions or conducting repetitive simulations, you may need to define several variables that are not straightforward manipulations of the task ID and file names. 

In those cases you may want to use a file that contains all the relevant information, and extract what you need for each run. 

As an example, let's say we want to break a variant calling run up over 100kb windows of the human genome. 

We can first define these 100kb windows using `bedtools`. 

```bash
module load bedtools

# a tab delimited file giving chromosome names and lengths for the human genome
GEN=/isg/shared/databases/alignerIndex/animal/hg38_ucsc/hg38_STAR/chrNameLength.txt

bedtools makewindows -g $GEN -w 100000 >100kb.win.bed
```

This bed file has 32489 lines and looks like this:

```bash
chr1	0	100000
chr1	100000	200000
chr1	200000	300000
chr1	300000	400000
chr1	400000	500000
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
#SBATCH --array=[1-32489]%20
##SBATCH --mail-type=ALL
##SBATCH --mail-user=YOUR.EMAIL@uconn.edu
#SBATCH -o %x_%A_%a.out
#SBATCH -e %x_%A_%a.err


echo "host name : " `hostname`
echo This is array job number $SLURM_ARRAY_TASK_ID

# put relevant data in variables
CHR=$(sed -n ${SLURM_ARRAY_TASK_ID}p 100kb.win.bed | cut -f 1)
START=$(expr $(sed -n ${SLURM_ARRAY_TASK_ID}p 100kb.win.bed | cut -f 2) + 1)
STOP=$(sed -n ${SLURM_ARRAY_TASK_ID}p 100kb.win.bed | cut -f 3)

# define region variable
REGION=${CHR}:${START}-${STOP}

echo This script will analyze region $REGION of the human genome. 

# subsequent lines would define input and output file names and the variant caller, e.g:

# module load freebayes
# freebayes -r $REGION -f ref.fa aln.bam >var.${REGION}.vcf

```

We specify the size of the job array as `#SBATCH --array=[1-32489]%20`. Note that it begins at 1 in this case. We use `sed` to print only the line of the file corresponding to the array task number and pipe that to `cut` to pull out the column that corresponds to the bit of data we want, placing that in a shell variable. Note that for `START`, we add 1 to the number in the bed file because a quirk of the BED format is that the start column is 0-indexed while the end column is 1-indexed. The region specification format of sequence:start-stop usually expects the start and stop positions to both be 1-indexed. 

This would result in 32489 jobs being run 20 at a time and produce as many output vcf files. The final step would probably be to use a package like `bcftools` or `vcflib` to filter and combine these into a single vcf. 

This approach could be generalized to any situation where you have a repetitive task you wish to parallelize, but input parameters that vary for each task. 

