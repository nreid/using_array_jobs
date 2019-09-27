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

For each task in the array, the variable SLURM_ARRAY_TASK_ID is set to the task number. These numbers can range from 0 to one less than the max number of jobs (on Xanadu this is ####), but can be any set of numbers or ranges, separated by commas. The numbers are specified on this line:

```bash
#SBATCH --array=[1-100]%20
```

The bracketed range gives the task numbers, and `%20` indicates that 20 tasks should run simultaneously. 

The other SLURM parameters should be set to match the requirements of each task. For example, if you are aligning a fastq file using `bwa` and you want to use 4 CPUs, you would set `#SBATCH -c 1`. 

The key to making these jobs useful is in how you use the SLURM_ARRAY_TASK_ID variable. As an example, let's make 26 empty files, each named A.txt, B.txt ...:

```bash
touch {A..Z}.txt
```

And say we wanted to use an array job to evaluate each of these files in parallel. One way to approach this is to create an array variable containing the list of files to analyze, and then use the SLURM_ARRAY_TASK_ID to retrieve elements of the list. 

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
FILES=($(ls -1 *.txt))

# get specific file name
	# note that FILES variable is 0-indexed so
	# for convenience we also began the task IDs with 0
FILENAME=${FILES[$SLURM_ARRAY_TASK_ID]}
echo $FILENAME

echo File $FILENAME was processed by task number $SLURM_ARRAY_TASK_ID on $(date) >>$FILENAME

```

Here we retrieve each `FILENAME` by using `SLURM_ARRAY_TASK_ID` to grab one element of `FILE`. 

Then we are simply writing that we processed FILENAME to the file. 

In this case our files all end in `.txt` etc, but we might want to write to `.out.txt`. We could do this by replacing `>>$FILENAME` with something like `>>$(echo $FILENAME | sed 's/.txt/out.txt/')`, but in that case, we should either write the output to a new directory or use output file naming that doesn't match pattern used to list the input files. This is because each array task generates the array variable `FILES` anew. If the results are written to the same directory and match the search pattern, then later tasks will try to run on the results files. 


