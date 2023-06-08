---
title: "Organising dependent Slurm jobs"
teaching: 10
exercises: 2
editor_options: 
  markdown: 
    wrap: 72
---

::: questions
-   How can I organise jobs that depend on each other?
:::

::: objectives
-   Know how to use the `--dependency` `sbatch` option
:::

## A pipeline to visualize `pi-cpu` results

In the previous episode, you learnt how to create job arrays to investigate
how the accuracy of `pi-cpu` changes with the number of trials used. You now
want to present these results to your supervisor graphically, but you still
need to aggregate and process the results further to create something presentable.
You decide that you want to show a bar chart where the X-axis corresponds to the
number of trials, and the Y-axis is the average absolute error calculated
determined from multiple calculations of $\pi$.

You want to test 7 different number of trials: 1E2 (100), 1E3 (1,000), ... , up to 1E8 (100,000,000).
These values are in `pi-process-input.txt`. These values are used inside `pi-process.sh`,
which is a script in `example-programs` that is a Slurm job array script like in
the job array episode. 

You know that you need to write a bit more code to complete the pipeline:

1. For each number of trials, the results from `pi-cpu` need to be averaged.
2. The individual averaged results need to be aggregated into a single file.

A diagram of the process looks like:

```
n = 1E2    1E3    ...     1E7     1E8
     |      |              |       |
     |      |              |       |
  pi-cpu  pi-cpu         pi-cpu  pi-cpu
     |      |              |       |
     |      |              |       |
    avg    avg            avg     avg
     |      |              |       |
     |______|___combine____|_______|
                results
                   |
                   |
                 final
                results
```

You decide that you want to do this with Slurm jobs because each step requires
different amounts of resources. But to set this up, you need to make use of:

## Slurm job dependencies

To setup Slurm jobs that are dependent on each other, you can make use of the
Slurm `--dependency`, or `-d` option. The syntax of the option is:

```bash
#SBATCH --dependency=condition:jobid   # or
#SBATCH -d condition:jobid
```

This will ask Slurm to ensure that the job that you're submitting doesn't start
until `condition` is satisfied for job `jobid`. For example, 

```bash
#SBATCH --dependency=afterok:1234
```

is requesting for the job you're submitting to start, only if job `1234` completes
successfully. This allows you to chain a series of jobs together, perhaps with
different resource requirements, without needing to monitor progress.

In your $\pi$ calculation scenario, submitting the second lot of array jobs (i.e., the `avg` jobs)
can be submitted using `--dependency=afterok:<jobid>`. However, this will mean that
*none* of the `avg` job array tasks will start until *all* of the `pi-cpu` jobs finish.
This results in lower job throughput as tasks wait around. 

We can achieve better job throughput by making use of the `aftercorr` condition
(short for "after corresponding"), which tells Slurm that each task in the second job
can start once the same task in the first job completes successfully.

## Setting up the `avg` array job

```bash
#!/bin/bash
# pi-postprocess.sh

#SBATCH --job-name=pi-avg
#SBATCH --output=%x-%a.out
#SBATCH --array=0-6
#SBATCH --cpus-per-task=2

./pi-avg pi-process-${SLURM_ARRAY_TASK_ID}.out
```

::: challenge

## Setting up the final `combine` job

HINT: you can use the `cut` utility to help you here. You might also need for loops

:::::::::::::

::: solution

Script could look like:

```bash
#!/bin/bash
# pi-combine.sh

#SBATCH --job-name=pi-combine
#SBATCH --output=%x.out

readarray -t niterations < pi-process-input.txt

for i in {0..6}
do
    echo ${niterations[$i]} $(cut -d ' ' -f 4 pi-avg-${i}.out)
done
```

::::::::::::

The driver program:

```bash
#!/bin/bash
# pi-driver.sh

process_jobid=$(sbatch --parsable pi-process.sh)
avg_jobid=$(sbatch --parsable --dependency=aftercorr:$process_jobid pi-avg.sh)
sbatch --dependency=afterok:${avg_jobid} pi-combine.sh
```

::: keypoints

-   Slurm job arrays are a great way to parallelise similar jobs!
-   The `SLURM_ARRAY_TASK_ID` environment variable is used to control individual array tasks' work
-   A file with all the parameters can be used to control array task parameters
-   `readarray` and `read` are useful tools to help you parse files. But it can also be done many other ways!

:::