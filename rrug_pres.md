RRUG - Going Parallel
========================================================
author: Matt McAnear
date: 2016-05-12
autosize: true


About Me
========================================================

- In Reno for ~1 year
- Math, Econ, and social science background
- I like R quite a bit.
- Focused on architecture in R
- Some mathematical modelling

About Clear Capital
==================================================
* AMC once based in Truckee, now in Reno
* Appraisals, Public Records, MLS Data; uniquely complete dataset for home prices
* Data Science team of 7 - looking for an eighth (Stat Analyst)

![img](cc.jpg)

Outline
========================================================

- Parallel Intro
- Forks
- PSOCK
- Clusters
- Lessons

Parallel
==================================================
* Base package in R
* Simplest way to get parallelizing
* Lots of functions come from the SNOW package, which is now deprecated
* `parLapply, parSapply, clusterApplyLB, clusterApply`...

Good Use Cases
==================================================
* Map-reduce algorithms
* Monte Carlo simulations
* Bootstrapping and replication
* Complex functions

Bad Use Cases
==================================================
* Iterative loops (output depends on previous step)
* Continuous edits required on an object
* Very simple, high speed, and/or low volume requests
* File Writes

Managing Expectations
==================================================
<font size="24">**Parallelism adds overhead.**</font>

Simplest Example
========================================================


```r
exampleData <- runif(100, min = 0, max = 100)
median(exampleData)
```

```
[1] 41.69822
```

Is the median accurate?
==================================================


```r
system.time(
    medians <- replicate(n = 50000, expr = {
        subSample <- sample(exampleData, size = length(exampleData), replace = TRUE)
        sampleMedian <- median(subSample)
    }, simplify = TRUE)
)
```

```
   user  system elapsed 
  2.178   0.034   2.219 
```

```r
mean(medians)
```

```
[1] 42.61138
```

Resulting Distribution
==================================================

![plot of chunk unnamed-chunk-3](rrug_pres-figure/unnamed-chunk-3-1.png)



Is the median accurate? - Parallel Edition
==================================================


```r
library(parallel)
system.time(
    medians <- mclapply(1:50000, function(run) {
        subSample <- sample(exampleData, size = length(exampleData), replace = TRUE)
        sampleMedian <- median(subSample)
    }, mc.cores = 4, mc.preschedule = TRUE)
)
```

```
   user  system elapsed 
  0.971   0.087   0.539 
```

```r
mean(unlist(medians))
```

```
[1] 42.63273
```

Same Distribution!
==================================================

![plot of chunk unnamed-chunk-5](rrug_pres-figure/unnamed-chunk-5-1.png)

What Happened?
==================================================
- `mclapply` creates multiple processes of R for operations and then returns the results
- Each child has access to all the objects in the parent environment, including global variables.
- *Implicit Parallelism*

Fork Clusters
==================================================

```r
cl <- makeForkCluster(nnodes = 4)
system.time(
    medians <- parLapply(cl = cl, X = 1:50000, function(run) {
        subSample <- sample(exampleData, size = length(exampleData), replace = TRUE)
        sampleMedian <- median(subSample)
    })
)
```

```
   user  system elapsed 
  0.023   0.005   0.531 
```

```r
stopCluster(cl)
```


Fork Clusters in a Nutshell
==================================================
* Very similar to just using mclapply
* Can't use `mclapply` within a cluster!
* Zombie processes...

![image](forkCluster.jpeg)


The (P)Sock Cluster
==================================================
* Not all of the "nodes" have to be one server!
* You have to export all the variables you want to use to the threads

![image](socks.jpg)


Clusters Compared
==================================================

mcLapply
- Can only be on one machine
- Implicit Object Sharing
- Implicit Parallelism

***

Fork Cluster
- Can only be on one machine
- Implicit Object Sharing
- Explicit Parallelism

***

PSOCK Cluster
- Multiple machines
- Explicit Object Sharing
- Explicit Parallelism

Split Strategy 1 - Create Cluster Across All your Cores
==================================================
* Does not have to be one machine (Advantage over Fork)
* Must copy data across machines - potentially a lot of overhead
* Maximum of 128 "nodes", i.e. 128 cores in a single cluster with PSOCK!


Split Strategy 2 - The Parent-Child-Grandchild Split (Map-Reduce)
==================================================
1. Divide the work into reasonable chunks and assign each chunk a key
2. Create a cluster with 1 node on each server. These are your first child processes.
3. On each parent process spawned by this cluster, create **n** (grand)child processes,
where **n** is the number of cores on the box (`detectCores()`).
4. Send necessary data to boxes and all clusters, both those with the child processes and grandchild processes
5. Apply function which uses the granchildren processes
6. Return results from grandchild processes to child processes
7. Return results from child processes to master process

Why not just make one cluster?
==================================================

- **128 node limit only applies to a single cluster object!**
- As long as you have less than 128 servers, you *should* be good.

![image](splitting_processes.gif)

Extended Clear Capital Example
==================================================
Main uses are data updates, model runs, and file parsing.

Caveats
==================================================
* Parallel is not always the answer.
* Parallelism can be difficult to debug.

![img](debugging.gif)


When is serial better?
==================================================
* Few iterations
* Low Complexity
* Vectorized vs. unvectorized code


Conclusion
==================================================

* Parallel is great for map-reduce sorts of problems.
* Clusters are great for managing servers, but can be finicky.
* Well written serial code is often more performant than poorly written
parallel code

<font size = "24">**Slow serial code is slow parallel code.**</font>

