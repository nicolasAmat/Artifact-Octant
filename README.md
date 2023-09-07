# Artifact for the Paper "Project and Conquer: Fast Quantifier Elimination for Checking Petri Nets Reachability"
----

Nicolas Amat, Silvano Dal Zilio, Didier Le Botlan

LAAS-CNRS, Université de Toulouse, CNRS, INSA, Toulouse, France

nicolas.amat@laas.fr


## Abstract

This artifact contains the tools and scripts for reproducing the experiments
found in our paper "Project and Conquer: Fast Quantifier Elimination for
Checking Petri Nets Reachability".

Our main tool, called Octant, can be used as a pre-processor for accelerating
the model-checking of generalized reachability properties on Petri nets. Given
an initial Petri net and an initial set of formulas, Octant takes advantage of
structural reductions in order to compute a set of simplified, projected
formulas that can be checked on a reduced Petri net while preserving the
verdicts.

## Requirements

To reproduce the experiments, the resources required are 4 CPU cores and an
amount of 16GB of RAM in order to reproduce similar conditions to those of the
Model Checking Contest.


## Tool Availability

Octant and SMPT are open-source (under GPLv3 license) tools respectively
available on GitHub at https://github.com/nicolasAmat/Octant and
https://github.com/nicolasAmat/SMPT.

External tools used in this artifact are all freely available:
- ITS-Tools and Tapaal: disk images with agreement of the authors from the [Model
  Checking Contest 2022](https://mcc.lip6.fr/2022/)
- [LoLA](https://theo.informatik.uni-rostock.de/theo-forschung/tools/lola/) (AGPLv3)
- [Barvinok](https://barvinok.sourceforge.io/) (GPLv3)
- [Tina toolbox](https://projects.laas.fr/tina/download.php) (freeware)
- [Latte](https://github.com/latte-int/latte-distro) (LiDIA)
- [z3](https://github.com/Z3Prover/z3) (MIT)


### Remark

To perform the evaluation of the state-of-the-art tools we dumped the disk
images of the Model Checking Contest 2022 that are publicly available at
https://mcc.lip6.fr/2022/results.php.


## 0 - Content of the artifact


The `Artifact/` directory is organized as follows:

```
-> run_experiments.sh                # script to run all the experiments

-> Dependencies/                     # dependencies
    -> Latte/                        # Latte distro (required by SMPT)
    -> Tina/                         # Tina toolbox (required by SMPT)
    -> z3/                           # SMT solver   (required by SMPT)

-> Tools/                            # tools directory
    -> Barvinok/                     # Barvinok                (evaluation on quantifier elimination)
    -> SMPT/                         # SMPT model checker      (evaluation on model checking)
    -> ITS-Tools/                    # ITS-Tools model checker (evaluation on model checking under real conditions)
    -> Tapaal/                       # Tapaal model checker    (evaluation on model checking under real conditions)
    -> LoLA/                         # LoLA model checker      (evaluation on model checking under real conditions)

-> Queries/                          # queries from the 2022 edition of the model checking contest
                                     # one subdirectory per instance containing:
                                     # the model (model.pnml), the queries (*-ReachabilityCardinality-*.xml),
                                     # and all the queries in one file (ReachabilityCardinality.xml)

-> Lists/                            # lists of instances and queries
    -> all_instances                 # all instances in the directory Queries/
    -> challenging_queries           # challenging queries (cf. paper)
    -> cex_queries                   # queries that are true EF phi or false AG phi
    -> inv_queries                   # queries that are false EF phi or true AG phi 
    -> picked_instances              # selection of instances (for the artifact experimentation)
    -> picked_queries                # selection of queries   (for the artifact experimentation)

-> Scripts/                          # scripts directory
    -> Manage_Benchmark/             # script to manage instances
        -> instances_of_queries.py   # generate the list of instances corresponding to a given list of queries

    -> Run/                          # scripts to run tools
        -> run_projections.sh        # compute the reduced net of a given instance and project all queries
        -> run_barvinok.sh           # project a query using Barvinok
        -> run_smpt.sh               # run SMPT (random walk or k-induction) on a given query
        -> run_itstools.sh           # run ITS-Tools on a given query
        -> run_tapaal.sh             # run Tapaal on a given query
        -> run_lola.sh               # run LoLA on a given query

-> Out2csv/                          # scripts to analyze logs in the Outputs/ directory
    -> projections.py                # generate CSV/projections.csv
    -> barvinok.py                   # generate CSV/barvinok.csv
    -> smpt.py                       # generate CSV/walk.csv and CSV/kinduction.csv
    -> model_checkers.py             # generate CSV/ITS-Tools.csv, CSV/Tapaal.csv and CSV/LoLA.csv
    
    -> Utils/
        -> pretty_csv.py             # script to pretty print a .csv file

-> CSV/                              # empty directory to store produced .csv files

-> Outputs/                          # empty directory to store raw outputs of the tools

-> tmp/                              # empty directory to store temporary files
```

## 1 - Installation

Build docker image:

```
$ docker build -t octant:latest .
```

Run docker image:
```
$ docker run -it octant:latest
```

### Apple Silicon

**Warning:** This artifact relies on x86_64 binaries. It is not possible to build it
natively on arm64 architecture.

Create a new cross-compiler builder:

```
$ docker buildx create --name mybuilder --driver docker-container --bootstrap
$ docker buildx use mybuilder
```

Build the docker image:
```
$ docker buildx build --platform linux/amd64 -t octant:latest --load .
```

Run the docker image:
```
$ docker run -it octant:latest
```


## 2 - Reproduce Experiments

Experiments can be automatically reproduced by running the following script:
```
$ ./run_experiments.sh
```

### Data analysis

Results are displayed on the standard output as they are computed. It is still
possible to analyze the raw outputs in the `Outputs/` directory, and the .csv
tables in the `CSV/` directory.

### Note

By default, the `run_experiments.sh` script computes projections for all the
queries of instances listed in `Lists/picked_instances` (there are 16 queries
for each instance), using Octant. Then we compute projections for the queries
listed in `Lists/picked_queries` using Barvinok, and proceed with model-checking
the queries on the same list (using both the original and projected versions).
It is possible to change the content of the two files  `Lists/picked_instances`
and `Lists/picked_queries`.

Reproducing all the experiments from the paper is not realistic without a
cluster of computers. For the model checking steps, we also set the time limit
to 60s instead of 180s in the paper. The time limits can be changed in the
`Scripts/Run/run_{tool}.sh` scripts by setting the variable `TIMEOUT`.

### Detailed description

The overall time to run the script is estimated at 1 h 10 min.

Our experiments are organized around 4 main steps and are based on a benchmark
borrowed from the 2022 edition of the [Model Checking Contest](https://mcc.lip6.fr/).
Petri net instances and queries are found in the folder `Queries/`.

- **Step 1: Reduction and projection (Octant) ~ 3 min**

We use Octant to compute projected queries and test if our projection is
complete. 
 
- **Step 2: Projections using Barvinok (for comparison) ~ 10min**

Query projection can be interpreted as an elimination of existentially
quantified variables in a Presburger formula. In this step, we compare our
results with a projection computed using the iscc tool, part of the Barvinok
numerical library.

- **Step 3: Random walk and k-induction ~ 15 min**

We use our symbolic model-checker, called SMPT, to compare the results computed
with queries listed in `Lists/picked_queries`, both on the initial and projected
system. We use two different methods: random walk (best suited to find
counter-examples) if the query is listed in the file `Lists/cex_queries` and
k-induction (best suited to prove invariants) if it is listed in file
`Lists/inv_queries`.

- **Step 4: Model checking under real conditions (ITS-Tools, Tapaal and LoLA) ~ 40min**

We again compare the results and computation time obtained when model-checking
queries before and after projection. This time we use three off-the-shelf
model-checkers that participated in the MCC: ITS-Tools, Tapaal and LoLA.

### Skipping experiments

You can comment some steps to skip them in the `run_experiments` script, but do
not skip "Step 1: Reduction and projection (Octant)" which generates files used
during the following steps (reduced net, projected formulas).
