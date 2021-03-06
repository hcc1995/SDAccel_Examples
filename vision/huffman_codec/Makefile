.PHONY: help

help::
	$(ECHO) "Makefile Usage:"
	$(ECHO) "  make all TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to generate the design for specified Target and Device."
	$(ECHO) ""
	$(ECHO) "  make clean "
	$(ECHO) "      Command to remove the generated non-hardware files."
	$(ECHO) ""
	$(ECHO) "  make cleanall"
	$(ECHO) "      Command to remove all the generated files."
	$(ECHO) ""
	$(ECHO) "  make check TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to run application in emulation."
	$(ECHO) ""
	$(ECHO) "  make build TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to build xclbin application."
	$(ECHO) ""
	$(ECHO) "  make run_nimbix DEVICE=<FPGA platform>"
	$(ECHO) "      Command to run application on Nimbix Cloud."
	$(ECHO) ""
	$(ECHO) "  make aws_build DEVICE=<FPGA platform>"
	$(ECHO) "      Command to build AWS xclbin application on AWS Cloud."
	$(ECHO) ""

# Points to Utility Directory
COMMON_REPO = ../../
ABS_COMMON_REPO = $(shell readlink -f $(COMMON_REPO))

TARGETS := hw
TARGET := $(TARGETS)
DEVICE := $(DEVICES)
XCLBIN := ./xclbin

include ./utils.mk

DSA := $(call device2sandsa, $(DEVICE))
BUILD_DIR := ./_x.$(TARGET).$(DSA)

BUILD_DIR_krnl_huffman = $(BUILD_DIR)/krnl_huffman

CXX := $(XILINX_SDX)/bin/xcpp
XOCC := $(XILINX_SDX)/bin/xocc

#Include Libraries
include $(ABS_COMMON_REPO)/libs/opencl/opencl.mk
include $(ABS_COMMON_REPO)/libs/logger/logger.mk
include $(ABS_COMMON_REPO)/libs/cmdparser/cmdparser.mk
include $(ABS_COMMON_REPO)/libs/simplebmp/simplebmp.mk
include $(ABS_COMMON_REPO)/libs/xcl2/xcl2.mk
CXXFLAGS += $(logger_CXXFLAGS) $(cmdparser_CXXFLAGS) $(simplebmp_CXXFLAGS) $(xcl2_CXXFLAGS)
LDFLAGS += $(logger_LDFLAGS) $(cmdparser_LDFLAGS) $(simplebmp_LDFLAGS) $(xcl2_LDFLAGS)
HOST_SRCS += $(logger_SRCS) $(cmdparser_SRCS) $(simplebmp_SRCS) $(xcl2_SRCS)
CXXFLAGS += $(opencl_CXXFLAGS) -Wall -O0 -g -std=c++14
LDFLAGS += $(opencl_LDFLAGS)

HOST_SRCS += src/main.cpp src/huffmancodec_optimized.cpp src/huffmancodec_naive.cpp src/bit_io.cpp src/huffmancodec_optimized_cpuonly.cpp
HOST_HDRS += src/bit_io.h src/huffmancodec_naive.h src/huffmancodec_optimized.h src/huffmancodec_optimized_cpuonly.h

# Host compiler global settings
CXXFLAGS += -fmessage-length=0
LDFLAGS += -lrt -lstdc++ 

# Kernel compiler global settings
CLFLAGS += -t $(TARGET) --platform $(DEVICE) --save-temps 
CLFLAGS += --xp param:compiler.version=31
CLFLAGS += --xp param:compiler.version=31


EXECUTABLE = huffman
CMD_ARGS = $(XCLBIN)/krnl_huffman.$(TARGET).$(DSA).xclbin -b ./data/rect_1024.bmp

EMCONFIG_DIR = $(XCLBIN)/$(DSA)

BINARY_CONTAINERS += $(XCLBIN)/krnl_huffman.$(TARGET).$(DSA).xclbin
BINARY_CONTAINER_krnl_huffman_OBJS += $(XCLBIN)/encode.$(TARGET).$(DSA).xo
BINARY_CONTAINER_krnl_huffman_OBJS += $(XCLBIN)/decode.$(TARGET).$(DSA).xo

CP = cp -rf
DATA = ./data

.PHONY: all clean cleanall docs emconfig
all: check-devices $(EXECUTABLE) $(BINARY_CONTAINERS) emconfig

.PHONY: exe
exe: $(EXECUTABLE)

.PHONY: build
build: $(BINARY_CONTAINERS)

# Building kernel
$(XCLBIN)/encode.$(TARGET).$(DSA).xo: src/krnl_huffman_singleptr.cl
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_krnl_huffman) -c -k encode -I'$(<D)' -o'$@' '$<'
$(XCLBIN)/decode.$(TARGET).$(DSA).xo: src/krnl_huffman_singleptr.cl
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_krnl_huffman) -c -k decode -I'$(<D)' -o'$@' '$<'
$(XCLBIN)/krnl_huffman.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_krnl_huffman_OBJS)
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_krnl_huffman) -l $(LDCLFLAGS) --nk encode:1 --nk decode:1 -o'$@' $(+)

# Building Host
$(EXECUTABLE): check-xrt $(HOST_SRCS) $(HOST_HDRS)
	$(CXX) $(CXXFLAGS) $(HOST_SRCS) $(HOST_HDRS) -o '$@' $(LDFLAGS)

emconfig:$(EMCONFIG_DIR)/emconfig.json
$(EMCONFIG_DIR)/emconfig.json:
	emconfigutil --platform $(DEVICE) --od $(EMCONFIG_DIR)

check: all
ifeq ($(TARGET),$(filter $(TARGET),sw_emu hw_emu))
	$(CP) $(EMCONFIG_DIR)/emconfig.json .
	XCL_EMULATION_MODE=$(TARGET) ./$(EXECUTABLE) $(XCLBIN)/krnl_huffman.$(TARGET).$(DSA).xclbin -b ./data/rect_1024.bmp
else
	 ./$(EXECUTABLE) $(XCLBIN)/krnl_huffman.$(TARGET).$(DSA).xclbin -b ./data/rect_1024.bmp
endif
ifneq ($(TARGET),$(findstring $(TARGET), sw_emu hw))
$(warning WARNING:Application supports only sw_emu hw TARGET. Please use the target for running the application)
endif

	sdx_analyze profile -i sdaccel_profile_summary.csv -f html

run_nimbix: all
	$(COMMON_REPO)/utility/nimbix/run_nimbix.py $(EXECUTABLE) $(CMD_ARGS) $(DSA)

aws_build: check-aws_repo $(BINARY_CONTAINERS)
	$(COMMON_REPO)/utility/aws/run_aws.py $(BINARY_CONTAINERS)

# Cleaning stuff
clean:
	-$(RMDIR) $(EXECUTABLE) $(XCLBIN)/{*sw_emu*,*hw_emu*} 
	-$(RMDIR) sdaccel_* TempConfig system_estimate.xtxt *.rpt
	-$(RMDIR) src/*.ll _xocc_* .Xil emconfig.json dltmp* xmltmp* *.log *.jou *.wcfg *.wdb

cleanall: clean
	-$(RMDIR) $(XCLBIN)
	-$(RMDIR) _x.*
	-$(RMDIR) ./decoded.bmp 
