Loop Fusion (CL)
======================

This example will demonstrate how to fuse two loops into one to improve the performance of an OpenCL kernel.

***KEY CONCEPTS:*** Kernel Optimization, Loop Fusion, Loop Pipelining

***KEYWORDS:*** xcl_pipeline_loop

## SUPPORTED PLATFORMS
Board | Software Version
------|-----------------
Xilinx Alveo U250|SDx 2018.3
Xilinx Alveo U200|SDx 2018.3
Xilinx Alveo U280 ES|SDx 2018.3
Xilinx Virtex UltraScale+ VCU1525|SDx 2018.3


##  DESIGN FILES
Application code is located in the src directory. Accelerator binary files will be compiled to the xclbin directory. The xclbin directory is required by the Makefile and its contents will be filled during compilation. A listing of all the files in this example is shown below

```
src/host.cpp
src/nearest_neighbor.cl
```

##  COMMAND LINE ARGUMENTS
Once the environment has been configured, the application can be executed by
```
./loop_fusion <nearest_neighbor XCLBIN>
```

