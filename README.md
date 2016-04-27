# DPT-OpenCV

##Basic Setup
Run `./setup.bsh` to download and install OpenCV, the testcases and setup the project (This downloads over a GB of data. It may take some time to complete)

Please note that cmake, make, build-essentials are all required for setup.bsh to run successfully ( `sudo apt-get install cmake make build-essentials` ). Everything documented here has only been tested on an Ubuntu 14.04.4 OS. It is not known if this would be replicable on anyother OS, unix-based or otherwise.

To compile and run the application:

`CMake .`

`make`

`./classify_images [directory containing images]`

classify_images should output which images have and which images do not contain faces (1 == face, 0 == not face)

##Profiling the software
The directory "profiling" provides two scripts "setup_profiling.bsh" and "remove_profiling.bsh" which, when executed from the "profiling" directory shall setup the application for profiling and revert this action respectively

"profiling/callgrind.out.11739" was produced using the "valgrind" tool on a subset of the positive and negative testcases (after profiling was setup using "setup_profiling.bsh" :

`valgrind --tool=callgrind ./classify_images [directory containing images]`

"profiling/callgraph.png" was generated using "qcachegrind" for Mac OS X (use "kcachegrind" for similar results on Linux)

The "modifiable_files" directory contains the files which shall be modified. 

##Exposing the parameters
In the "modifiable_files directory there is an "expose_integers.bsh" script which when run can extract the integer constants from a program. The usage of the script is as follows:

`./expose_integers.bsh [input file] [the output file] [define_file]`

This script utalises the "replace_integer.pl" script

We used this create the "cascadedetect_exposed.cpp" and "cascadedetect_exposed.hpp" files along with the "replaces.hpp" file which contains all of the extracted integer constants. We add this as an include at the top of the file.

These files are copied into the appropriate directory during setup. As with the profiling directory, the scripts and data are mostly for reproducability purposes.

##Filtering the extracted constants
For optimisation we want to remove all the constants that are of low value or of no use at all. Since, for this example, we have over 500 constants we wrote a script which goes through each constant and runs some basic tests to indicate whether this is worthwhile.

The "run_sensativity_filteration.bsh" script works by iterating through each constant. For each one, it first of all increments it by one. If it does not compile or produces an outcome which crashes on the "sensitivity_set" then this is considered too sensative to really be of value. If it passes this test then the integer constant has 50 added. If the program compiles, runs, and completes running in the normal amount of time (within the 95% confidence interval of the original application) for the "training_st" then we determine this constant to not be sensative enough for our requirements (it may not be necessisary at all!). The goal is to find constants which can be modified slightly without crashing but change the running of the software when modified by a small amount. A sensativity "sweet-spot" as it were.

The output of the "run_sensativity_filteration.bsh" script can be interpreted as follows (it is in a CSV format):

<Constant>,<Not_too_sensative?>,<time_training_set>,<sensative_enough?>

We modify only those constants where <Not_too_sensative?> and <sensative_enough> are true. `cat run_sensativity_filteration_output.csv | awk -F "," '($2=="true" && $4=="true"){print $1}' >replaces_selection.dat`

This replaces_selection.dat is then used in the final parameter tuning step

##Running Deep Parameter Tuning
The MOEA framework (2.9) is used to run the NSGA-II algorithm. This should have been downloaded and setup with the "setup.bsh" script was run. For this investigation a setup is provided: "DeepParameterTuning.java". This is very much hard-coded to OpenCV. When executed it will run for 10 generations with a population size of 100. The initial generation is seeded with the Deep Parameters in their original state and variants within the local-neighbourhood. To compile execute the following:

`javac -cp ".:MOEAFramework-2.9/lib/*" DeepParameterTuning.java`

To Run:

`java -cp ".:MOEAFramework-2.9/lib/*" DeepParameterTuning`

##Results
The output running this on an Ubuntu 14.04.4 m4.large Amazon EC2 Instance (2x2.4GHz Intel Xeon E5-2676 v3 processor, 8GiB of memory, SSD Storage) can be found in the "100ind_10gen_run" directory

The "100ind_10gen_run" directory contains the following files:

DeepParameterTuning_100ind_10gen.dat -> the raw output of executing DeepParameterTuning.java

no_mod_100_run.csv -> the output of running classify_images 100 times using "fitness_function.bsh"

original_output.dat -> the average of "no_mod_100_run.csv"

pareto_optimal.csv -> The Pareto Optimal solutions, derived from "DeepParameterTuning_100ind_10gen.dat"

time_correctness_pareto_front.pdf -> The Pareto Optimal solutions from "pareto_optimal.csv" plotted. Includes "original_output.dat" to show the original appplication (in blue as opposed to the Pareto Frontier in RED)
