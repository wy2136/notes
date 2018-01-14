# CESM 1.2.1 Porting on Tiger

### Download the Code
Get an account at http://www.cesm.ucar.edu/models/register/register.html and download the source code:
	
	cd /tigress/$USER
	svn co https://svn-ccsm-models.cgd.ucar.edu/cesm1/release_tags/cesm1_2_1

We also need extra downloads due to a reported bug (see https://bb.cgd.ucar.edu/googlecode-repositories-are-offline-pio-source-not-found):

	svn export --force https://github.com/PARALLELIO/genf90/tags/genf90_140121 ./cesm1_2_1/tools/cprnc/genf90
	svn export --force https://github.com/NCAR/ParallelIO.git/tags/pio1_7_1/pio ./cesm1_2_1/models/utils/pio

### Create an Experiment Case
Go to the `scripts` directory and create an experiment case (`$CESMROOT` is `cesm1_2_1`, where the source files are downloaded):

    cd $CESMROOT/scripts
    ./create_newcase -case test1 -res f45_g37 -compset X -mach userdefined

### Configure
    
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

### Build

    ./test1.build
	
### Run
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

and the line
	
	srun -n $npes $EXEROOT/cesm.exe >&! cesm.log.$LID

after the line

	#mpirun -np 16 $EXEROOT/cesm.exe >&! cesm.log.$LID

Run the model:
	
	./test1.run
