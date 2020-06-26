SHELL := /bin/bash
OUTPUT_DIR ?= .

SIZE ?= SMALL
SECURITY ?= false
TSO_MM ?= false
STORE_PREFETCH ?= false
SELF_INV_CACHE ?=

# check RISCY
ifeq ($(RISCY_HOME),)
$(error Please set up the RISCY environment first (source setup.sh))
endif

SYNC_CMD := rsync -r --delete

BUILD_DIR := $(OUTPUT_DIR)/build_$(SIZE)
VERILOG_DIR := $(OUTPUT_DIR)/verilog_$(SIZE)
TARGET_DIR := $(OUTPUT_DIR)/design_$(SIZE)
TARGET_VERILOG_DIR := $(TARGET_DIR)/verilog
TARGET_BSV_DIR := $(TARGET_DIR)/bsv

BSV_PATH := $(RISCY_HOME)/procs/lib \
		    $(RISCY_HOME)/procs/RV64G_OOO \
		    $(RISCY_HOME)/coherence/src

TARGET_BSV_PATH := $(foreach path,$(BSV_PATH),$(subst $(RISCY_HOME),$(TARGET_BSV_DIR),$(path)))

OPT_BSV_PATH := $(foreach path,$(BSV_PATH),$(addprefix -p +:,$(path)))

OPT_FLAGS = -unspecified-to X \
			-opt-undetermined-vals

BSV_DEF = -D CORE_$(SIZE) \
		  -D CACHE_LARGE \
		  -D NUM_CORES=1 \
		  -D L1_CACHE_LRU \
		  -D XILINX_FP_FMA_LATENCY=3 \
		  -D XILINX_INT_MUL_LATENCY=2
# use Xilinx FPU so we can override FPU
BSV_DEF += -D USE_XILINX_FPU

# security
ifeq ($(SECURITY),true)
BUILD_DIR := $(BUILD_DIR)_security
VERILOG_DIR := $(VERILOG_DIR)_security
BSV_DEF += -D SECURITY
endif

# self inv cache
ifneq ($(SELF_INV_CACHE),)
BUILD_DIR := $(BUILD_DIR)_si$(SELF_INV_CACHE)
VERILOG_DIR := $(VERILOG_DIR)_si$(SELF_INV_CACHE)
BSV_DEF += -D SELF_INV_CACHE -D L1D_MAX_HITS=$(SELF_INV_CACHE)
endif

# TSO
ifeq ($(TSO_MM),true)
BUILD_DIR := $(BUILD_DIR)_tso
VERILOG_DIR := $(VERILOG_DIR)_tso
BSV_DEF += -D TSO_MM
ifeq ($(STORE_PREFETCH),true)
BUILD_DIR := $(BUILD_DIR)sp
VERILOG_DIR := $(VERILOG_DIR)sp
BSV_DEF += -D STORE_PREFETCH
endif
endif

COMMAND = bsc -u \
		  -bdir $(BUILD_DIR) \
		  -info-dir $(BUILD_DIR) \
		  -simdir $(BUILD_DIR) \
		  -vdir $(VERILOG_DIR) \
		  -aggressive-conditions \
		  -check-assert \
		  -suppress-warnings S0073:G0024 \
		  +RTS -K1G -RTS \
		  -steps-max-intervals 200 \
		  $(OPT_BSV_PATH) \
		  $(OPT_FLAGS) \
		  $(BSV_DEF) \
		  -verilog CoreTop.bsv

verilog: 
	mkdir -p $(BUILD_DIR)
	mkdir -p $(VERILOG_DIR) 
	echo $(COMMAND) > $(BUILD_DIR)/log.txt
	eval $(COMMAND) 2>& 1 | tee -a $(BUILD_DIR)/log.txt

pack-design: verilog $(TARGET_BSV_PATH) $(TARGET_VERILOG_DIR)
	tar zcf design.tar.gz $(TARGET_DIR)

$(TARGET_BSV_PATH): $(TARGET_DIR)/bsv/%: $(RISCY_HOME)/%
	@echo "$< ==> $@"
	@[[ -e $< ]] || (echo "some error for $<" && false)
	@mkdir -p $(dir $@)
	@$(SYNC_CMD) $</* $@

$(TARGET_VERILOG_DIR): $(VERILOG_DIR)
	@echo "$< ==> $@"
	@[[ -e $< ]] || (echo "some error for $<" && false)
	@mkdir -p $(dir $@)
	@$(SYNC_CMD) $</* $@

clean:
	rm -rf $(BUILD_DIR) $(VERILOG_DIR) $(TARGET_DIR) design.tar.gz

.PHONY: clean $(TARGET_BSV_PATH) $(TARGET_VERILOG_DIR)

