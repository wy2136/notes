# CESM 1.2.1 Porting on Tiger
* Wenchang Yang (wenchang@princeton.edu)
* Department of Geosciences, Princeton University

**Acknowledgement**: Many thanks to Dr. [**Angel Munoz**](https://scholar.princeton.edu/agmunoz) who shares his great expertise on climate modeling and makes this note possible.

### Step 1: Download the Code
Get an account at http://www.cesm.ucar.edu/models/register/register.html and download the source code:
	
	cd /tigress/$USER
	svn co https://svn-ccsm-models.cgd.ucar.edu/cesm1/release_tags/cesm1_2_1

We also need extra downloads due to a reported bug (see https://bb.cgd.ucar.edu/googlecode-repositories-are-offline-pio-source-not-found):

	svn export --force https://github.com/PARALLELIO/genf90/tags/genf90_140121 ./cesm1_2_1/tools/cprnc/genf90
	svn export --force https://github.com/NCAR/ParallelIO.git/tags/pio1_7_1/pio ./cesm1_2_1/models/utils/pio

### Step 2: Create an Experiment Case
Go to the `scripts` directory and create an experiment case (`$CESMROOT` is `cesm1_2_1`, where the source files are downloaded):

    cd $CESMROOT/scripts
    ./create_newcase -case test1 -res f45_g37 -compset X -mach userdefined

### Step 3: Set up the Configuration
    
Edit `env_*.xml` files using the tool of `xmlchange`:

    ./xmlchange OS=Linux
    ./xmlchange COMPILER=gnu
    ./xmlchange MPILIB=openmpi
    ./xmlchange EXEROOT=/scratch/gpfs2/$USER/cesm1_2_1/test1
    ./xmlchange RUNDIR=/scratch/gpfs2/$USER/cesm1_2_1/test1
    ./xmlchange DIN_LOC_ROOT=/tigress/$USER/cesm1_2_1/inputdata
    ./xmlchange MAX_TASKS_PER_NODE=16

Set up the experiment:
    
    cd test1
    ./cesm_setup

Edit `Macro`:
	
	vi Macro

in line 9

    SLIBS+=# USERDEFINED $(shell $(NETCDF_PATH)/bin/nc-config --flibs)
    --->
    SLIBS+= $(LDFLAGS) -lnetcdff -lnetcdf

in line 19
    
    FFLAGS:= -O -fconvert=big-endian -ffree-line-length-none -ffixed-line-length-none
    ---->    
    FFLAGS:= -O -fconvert=big-endian -ffree-line-length-none -ffixed-line-length-none -fno-range-check -fcray-pointer
    
in line 37
    
    NETCDF_PATH:= USERDEFINED_MUST_EDIT_THIS
    ---->
    NETCDF_PATH:= $(NETCDF) /usr/local/netcdf/gcc/hdf5-1.8.12/4.3.1.1
    
Edit `Tools/Makefile`
	
	vi Tools/Makefile

in line 286

    NETCDF_PATH=$(NETCDF_PATH) LDFLAGS="$(LDFLAGS)" \
    ---->    
    NETCDF_PATH="$(NETCDF_PATH)" LDFLAGS="$(LDFLAGS)" \

Edit `env_mach_specific`:
	
	vi env_mach_specific

add the following lines:
    
    source /usr/share/Modules/init/csh
    module load openmpi/gcc/2.0.2/64
    module load netcdf/gcc/hdf5-1.8.12/4.3.1.1
    module load hdf5/gcc/1.8.12

### Step 4: Build

    ./test1.build
	
### Step 5: Run
Edit the `env_run.xml` by using the `xmlchange` tool:
	
    ./xmlchange BATCHQUERY=squeue
    ./xmlchange BATCHSUBMIT=sbatch

Edit `test1.run` by adding the following lines after the first line:
	
	#SBATCH -N 16 # node count
	#SBATCH --ntasks-per-node=16
	#SBATCH -t 00:30:00
	# sends mail when process begins, and
	# when it ends. Make sure you define your email
	#SBATCH --mail-type=begin
	#SBATCH --mail-type=end
	#SBATCH --mail-user=yournetid@princeton.edu
	set npes = 256

and add the line
	
	srun -n $npes $EXEROOT/cesm.exe >&! cesm.log.$LID

after the line

	#mpirun -np 16 $EXEROOT/cesm.exe >&! cesm.log.$LID

Run the model:
	
	sbatch test1.run
	
### FLORish
After running `test1` successfully, we can try another experiment case called `FLORish`, which has a nontrivial configuration and a spatial resolution similar to the [FLOR](https://www.gfdl.noaa.gov/cm2-5-and-flor/) model from GFDL .

	cd $CESMROOT/scripts
	./create_newcase -case ../cases/FLORish -res f05_g16 -compset B1850CN -mach userdefined

For FLORish, we need to create the `ice/cice` directory and download required data before building and running:
	
	cd $CESMROOT/inputdata
	mkdir ice
	mkdir ice/cice
	cd ice/cice
	svn export https://svn-ccsm-inputdata.cgd.ucar.edu/trunk/inputdata/ice/cice/iced.0001-01-01.gx1v6_20080212

Otherwise, the configuring and building process are similar to the case `test1` as described above (step 2-4). The running configuration is a little different:

First, edit the `env_run.xml` file to change the running period from 5 days to 3 years:
	
	./xmlchange STOP_OPTION=nyears
	./xmlchange STOP_N=3
 
Second, edit the `FLORish.run` script:
	
	#SBATCH -N 16 # node count
	---->
	#SBATCH -N 32 # node count
	
	
	#SBATCH -t 00:30:00
	---->
	#SBATCH -t 20:00:00
	
	
	set npes = 256
	---->
	set npes = 512
	
Now, we can run the model:
	
	sbatch FLORish.run
	
Good luck!

### Define the Machine `tiger`
It is great to enable the out-of-the-box capability for the machine `tiger` so that most of the work in step 3 and 5 can be avoided if we specify the ``-mach` option in step 2 as `tiger`(or whatever name you want) instead of `userdefined`. Motivated by the `cesm1.2.1` [user guide](http://www.cesm.ucar.edu/models/cesm1.2/cesm/doc/usersguide/x1794.html), our implementation is:

Step 1, pick the name `tiger`, or whatever name you like. And go to the directory `$CESMROOT/scripts/ccsm_utils/Machines`
	
	cd $CESMROOT/scripts/ccsm_utils/Machines

Step 2, Edit `config_machines.xml` and add a section for `tiger`. Copy `userdefined` content and modify:
	
	<machine MACH="tiger">
	        <DESC>Tiger at Princeton, Linux, slurm, 16 pes/node, gnu/openmpi/netcdf</DESC> <!-- can be anything -->
	        <OS>LINUX</OS>              <!-- LINUX,Darwin,CNL,AIX,BGL,BGP -->
	        <COMPILERS>gnu</COMPILERS>  <!-- intel,ibm,pgi,pathscale,gnu,cray,lahey -->
	        <MPILIBS>openmpi</MPILIBS>  <!-- openmpi, mpich, ibm, mpi-serial -->
	        <RUNDIR>/scratch/gpfs2/$CCSMUSER/cesm1_2_1/$CASE/run</RUNDIR>       <!-- complete path to the run directory -->
	        <EXEROOT>/scratch/gpfs2/$CCSMUSER/cesm1_2_1/$CASE/bld</EXEROOT>     <!-- complete path to the build directory -->
	        <DIN_LOC_ROOT>/tigress/yourNetID/cesm1_2_1/inputdata</DIN_LOC_ROOT>  <!-- complete path to the inputdata directory -->
	        <DIN_LOC_ROOT_CLMFORC>USERDEddFINED_optional_build</DIN_LOC_ROOT_CLMFORC> <!-- path to the optional forcing data for CLM (for CRUNCEP forcing) -->
	        <DOUT_S>FALSE</DOUT_S>                                            <!-- logical for short term archiving -->
	        <DOUT_S_ROOT>USERDEFINED_optional_run</DOUT_S_ROOT>               <!-- complete path to a short term archiving directory -->
	        <DOUT_L_MSROOT>USERDEFINED_optional_run</DOUT_L_MSROOT>           <!-- complete path to a long term archiving directory -->
	        <CCSM_BASELINE>USERDEFINED_optional_run</CCSM_BASELINE>           <!-- where the cesm testing scripts write and read baseline results -->
	        <CCSM_CPRNC>USERDEFINED_optional_test</CCSM_CPRNC>                <!-- path to the cprnc tool used to compare netcdf history files in testing -->
	        <BATCHQUERY>squeue</BATCHQUERY>
	        <BATCHSUBMIT>sbatch</BATCHSUBMIT>
	        <SUPPORTED_BY>USERDEFINED_optional</SUPPORTED_BY>
	        <GMAKE_J>1</GMAKE_J>
	        <MAX_TASKS_PER_NODE>16</MAX_TASKS_PER_NODE>
	</machine>
Replace `yourNetID` with your Princeton NetID here.

Step 3, edit `config_compilers.xml` to translate the additions you made to the Macros file to support `tiger` specific settings.
	
	<compiler COMPILER="gnu" MACH="tiger">
	  <SLIBS>$(LDFLAGS) -lnetcdff -lnetcdf</SLIBS>
	  <ADD_FFLAGS> -fno-range-check -fcray-pointer </ADD_FFLAGS>
	  <NETCDF_PATH>/usr/local/netcdf/gcc/hdf5-1.8.12/4.3.1.1</NETCDF_PATH>
	  <PNETCDF_PATH></PNETCDF_PATH>
	  <ADD_CPPDEFS></ADD_CPPDEFS>
	  <CONFIG_ARGS></CONFIG_ARGS>
	  <ESMF_LIBDIR></ESMF_LIBDIR>
	  <MPI_LIB_NAME></MPI_LIB_NAME>
	  <MPI_PATH></MPI_PATH>
	</compiler>

Step 4, create an `env_mach_specific.tiger` file. This should be a copy of the `env_mach_specific` file from the `test1` case directory.
	
	cp $CCSMROOT/scripts/test1/env_mach_specific $CCSMROOT/scripts/ccsm_utils/Machines/env_mach_specific.tiger

Step 5, create an `mkbatch.tiger` file. Copy the `mkbatch.userdefined` file to `mkbatch.tiger`
	
	cp mkbatch.userdefined mkbatch.tiger

and add the following lines to `mkbatch.tiger` after line 30 of `#!/bin/csh -f`:
	
	#!/bin/csh -f
	#SBATCH -N 32 # node count
	#SBATCH --ntasks-per-node=16
	#SBATCH -t 20:00:00
	# sends mail when process begins, and
	# when it ends. Make sure you define your email
	#SBATCH --mail-type=begin
	#SBATCH --mail-type=end
	#SBATCH --mail-user=yourNetID@princeton.edu
	set npes = 512

That's it! Next time the build/run process will be much easier:
	
	cd $CESMROOT/scripts
	./create_newcase -case test_tiger -res f45_g37 -compset X -mach tiger
	
	cd test_tiger
	./cesm_setup
	
	./test_tiger.build

Edit the `test_tiger.run` file. Replace the yourNetID with your NetID and change the line of `mpirun` to `srun -n $npes $EXEROOT/cesm.exe >&! cesm.log.$LID`. You might also need to change the node count or SBATCH time option depending the nature of your case. After these have all been done, we can run the model:
	
	sbatch test_tiger.build


