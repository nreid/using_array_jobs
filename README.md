# Job Arrays on Xanadu

A guide to splitting up big jobs using slurm job arrays on UConn's Xanadu cluster. 

## What is a job array?

According to the [SLURM](https://slurm.schedmd.com/job_array.html) documentation:

>Job arrays offer a mechanism for submitting and managing collections of similar jobs quickly and easily

That is, if you want to do something very similar many times, you can split the job up into an array of smaller jobs to parallelize it. 

Some examples of situations where array jobs can be helpful:

- Aligning many fastq files to a reference genome.
- Evaluating many positions in the genome (e.g. variant calling). 
- Simulating and evaluating many datasets. 

