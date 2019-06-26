# Experiment of increasing CO2 by 1% annually until doulbed

1. create_newcase
  
    create_newcase -case cases/TCR_B1850CN_f05g16_FLORish_tigercpu_intelmpi18_512PE -res f05_g16 -compset B1850CN -mach tigercpu -pes_file wy_pes_file_FLORish_512PE.xml
  
2. set up branch run

    ./xmlchange RUN_TYPE=branch
    ./xmlchange RUN_REFCASE=B1850CN_f05g16_FLORish_tigercpu_intelmpi18_512PE
    ./xmlchange RUN_REFDATE=0101-01-01
