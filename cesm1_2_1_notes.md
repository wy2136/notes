# CESM 1.2.1 Notes

### Download
    svn co https://svn-ccsm-models.cgd.ucar.edu/cesm1/release_tags/cesm1_2_1

    svn export --force https://github.com/PARALLELIO/genf90/tags/genf90_140121 ./cesm1_2_1/tools/cprnc/genf90
    svn export --force https://github.com/NCAR/ParallelIO.git/tags/pio1_7_1/pio ./cesm1_2_1/models/utils/pio

### Build

    ./create_newcase -case test1 -res f45_g37 -compset X -mach userdefined
    
##### Edit xml files:

    ./xmlchange OS=Linux
    ./xmlchange COMPILER=gnu
    ./xmlchange MPILIB=openmpi
    ./xmlchange EXEROOT=/scratch/gpfs2/$USER/cesm1_2_1/test1
    ./xmlchange RUNDIR=/scratch/gpfs2/$USER/cesm1_2_1/test1
    ./xmlchange DIN_LOC_ROOT=/tigress/wenchang/cesm1_2_1/inputdata
    ./xmlchange MAX_TASKS_PER_NODE=16

    ./cesm_setup

##### Edit Macro File
    
    SLIBS+=# USERDEFINED $(shell $(NETCDF_PATH)/bin/nc-config --flibs)
to    
    
    SLIBS+= $(LDFLAGS) -lnetcdff -lnetcdf
    
    
    FFLAGS:= -O -fconvert=big-endian -ffree-line-length-none -ffixed-line-length-none
to   
    
    FFLAGS:= -O -fconvert=big-endian -ffree-line-length-none -ffixed-line-length-none -fno-range-check -fcray-pointer
    
    
    NETCDF_PATH:= USERDEFINED_MUST_EDIT_THIS
to    
    
    NETCDF_PATH:= $(NETCDF) /usr/local/netcdf/gcc/hdf5-1.8.12/4.3.1.1
    
##### Edit Tools/Makefile

    NETCDF_PATH=$(NETCDF_PATH) LDFLAGS="$(LDFLAGS)" \
to
    
    NETCDF_PATH="$(NETCDF_PATH)" LDFLAGS="$(LDFLAGS)" \
