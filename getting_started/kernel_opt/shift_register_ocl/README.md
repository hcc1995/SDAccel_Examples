Shift Register (CL)
======================

This example demonstrates how to shift values in registers in each clock cycle

***KEY CONCEPTS:*** Kernel Optimization, Shift Register, FIR

***KEYWORDS:*** xcl_array_partition, getprofilingInfo()

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
src/fir.cl
src/host.cpp
```

##  COMMAND LINE ARGUMENTS
Once the environment has been configured, the application can be executed by
```
./shift_register <fir XCLBIN>
```

