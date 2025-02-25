# ==============================================================================
# Uncomment or add the design to run
# ==============================================================================
default: finish
.DEFAULT_GOAL := finish
PLATFORM_ARG  ?= sky130hd
DESIGN_CONFIG = ./design/$(PLATFORM_ARG)/cryo/config.mk
SPICE_FILE    ?= sky130_fd_sc_hd.spice

# ==============================================================================
#  ____  _____ _____ _   _ ____
# / ___|| ____|_   _| | | |  _ \
# \___ \|  _|   | | | | | | |_) |
#  ___) | |___  | | | |_| |  __/
# |____/|_____| |_|  \___/|_|
#
# ==============================================================================

#-------------------------------------------------------------------------------
# Default target when invoking without specific target.
.DEFAULT_GOAL := finish

#-------------------------------------------------------------------------------
# Proper way to initiate SHELL for make
SHELL          = /bin/bash
.SHELLFLAGS    = -o pipefail -c

#-------------------------------------------------------------------------------
# Setup variables to point to root / head of the OpenROAD directory
# - the following settings allowed user to point OpenROAD binaries to different
#   location
# - default follows OpenFASOC's directory structure, with absolute path to flow/
export FLOW_HOME ?= $(shell pwd)
export COMMON_HOME ?= $(FLOW_HOME)/../../../common

#-------------------------------------------------------------------------------
# Setup variables to point to other location for the following sub directory
# - designs - default is under current directory
# - platforms - default is under current directory
# - work home - default is current directory
# - utils, scripts, test - default is under current directory
export DESIGN_HOME   ?= $(FLOW_HOME)/design
export PLATFORM_HOME ?= $(COMMON_HOME)/platforms
export WORK_HOME     ?= $(shell pwd)

export UTILS_DIR     ?= $(FLOW_HOME)/util
export SCRIPTS_DIR   ?= $(FLOW_HOME)/scripts
export TEST_DIR      ?= $(FLOW_HOME)/test

#-------------------------------------------------------------------------------
# Include design and platform configuration
include $(DESIGN_CONFIG)

PUBLIC=nangate45 sky130hd sky130hs asap7

ifneq ($(wildcard $(PLATFORM_HOME)/$(PLATFORM)),)
  export PLATFORM_DIR = $(PLATFORM_HOME)/$(PLATFORM)
else ifneq ($(findstring $(PLATFORM),$(PUBLIC)),)
  export PLATFORM_DIR = ./platforms/$(PLATFORM)
else ifneq ($(wildcard ../../$(PLATFORM)),)
  export PLATFORM_DIR = ../../$(PLATFORM)
else
  $(error [ERROR][FLOW] Platform '$(PLATFORM)' not found.)
endif

$(info [INFO][FLOW] Using platform directory $(PLATFORM_DIR))
include $(PLATFORM_DIR)/config.mk

export GALLERY_REPORT ?= 0
# Enables hierarchical yosys
export SYNTH_HIERARCHICAL ?= 0
export SYNTH_STOP_MODULE_SCRIPT = $(OBJECTS_DIR)/mark_hier_stop_modules.tcl
ifeq ($(SYNTH_HIERARCHICAL), 1)
export HIER_REPORT_SCRIPT = $(SCRIPTS_DIR)/synth_hier_report.tcl
export MAX_UNGROUP_SIZE ?= 0
endif
# Enables Re-synthesis for area reclaim
export RESYNTH_AREA_RECOVER ?= 0
export RESYNTH_TIMING_RECOVER ?= 0
export ABC_AREA ?= 0

# Global setting for Synthesis
export SYNTH_ARGS ?= -flatten

# Global setting for Floorplan
export PLACE_PINS_ARGS

#export FLOW_VARIANT ?= base

export GPL_TIMING_DRIVEN ?= 1
export GPL_ROUTABILITY_DRIVEN ?= 1

export ENABLE_DPO ?= 1
export DPO_MAX_DISPLACEMENT ?= 5 1

# Setup working directories
export DESIGN_NICKNAME ?= $(DESIGN_NAME)

export DESIGN_DIR  = $(dir $(DESIGN_CONFIG))
export LOG_DIR     = $(WORK_HOME)/logs/$(PLATFORM)/$(DESIGN_NICKNAME)/$(FLOW_VARIANT)
export OBJECTS_DIR = $(WORK_HOME)/objects/$(PLATFORM)/$(DESIGN_NICKNAME)/$(FLOW_VARIANT)
export REPORTS_DIR = $(WORK_HOME)/reports/$(PLATFORM)/$(DESIGN_NICKNAME)/$(FLOW_VARIANT)
export RESULTS_DIR = $(WORK_HOME)/results/$(PLATFORM)/$(DESIGN_NICKNAME)/$(FLOW_VARIANT)

ifdef BLOCKS
  $(info [INFO][FLOW] Invoked hierarchical flow.)
  $(foreach block,$(BLOCKS),$(info Block ${block} needs to be hardened.))
  $(foreach block,$(BLOCKS),$(eval BLOCK_CONFIGS += ./designs/$(PLATFORM)/$(DESIGN_NICKNAME)/$(block)/config.mk))
  $(foreach block,$(BLOCKS),$(eval BLOCK_LEFS += ./results/$(PLATFORM)/$(DESIGN_NICKNAME)_$(block)/$(FLOW_VARIANT)/${block}.lef))
  $(foreach block,$(BLOCKS),$(eval BLOCK_GDS += ./results/$(PLATFORM)/$(DESIGN_NICKNAME)_$(block)/$(FLOW_VARIANT)/6_final.gds))
  $(foreach block,$(BLOCKS),$(eval BLOCK_CDL += ./results/$(PLATFORM)/$(DESIGN_NICKNAME)_$(block)/$(FLOW_VARIANT)/6_final.cdl))
  export ADDITIONAL_LEFS += $(BLOCK_LEFS)
  export ADDITIONAL_GDS += $(BLOCK_GDS)
  export GDS_FILES += $(BLOCK_GDS)
  ifdef CDL_FILES
    export CDL_FILES += $(BLOCK_CDL)
  endif
endif

export RTLMP_RPT_DIR ?= $(OBJECTS_DIR)/rtlmp
export RTLMP_RPT_FILE ?= partition.txt
export RTLMP_BLOCKAGE_FILE ?= $(OBJECTS_DIR)/rtlmp/partition.txt.blockage

#-------------------------------------------------------------------------------
ifeq (, $(strip $(NPROC)))
  # Linux (utility program)
  NPROC := $(shell nproc 2>/dev/null)

  ifeq (, $(strip $(NPROC)))
    # Linux (generic)
    NPROC := $(shell grep -c ^processor /proc/cpuinfo 2>/dev/null)
  endif
  ifeq (, $(strip $(NPROC)))
    # BSD (at least FreeBSD and Mac OSX)
    NPROC := $(shell sysctl -n hw.ncpu 2>/dev/null)
  endif
  ifeq (, $(strip $(NPROC)))
    # Fallback
    NPROC := 1
  endif
endif
export NUM_CORES := $(NPROC)

export LSORACLE_CMD ?= $(shell command -v lsoracle)
ifeq ($(LSORACLE_CMD),)
  LSORACLE_CMD = $(abspath $(FLOW_HOME)/../tools/install/LSOracle/bin/lsoracle)
endif

LSORACLE_PLUGIN ?= $(abspath $(FLOW_HOME)/../tools/install/yosys/share/yosys/plugin/oracle.so)
export LSORACLE_KAHYPAR_CONFIG ?= $(abspath $(FLOW_HOME)/../tools/install/LSOracle/share/lsoracle/test.ini)
ifneq ($(USE_LSORACLE),)
  YOSYS_FLAGS ?= -m $(LSORACLE_PLUGIN)
endif

YOSYS_FLAGS += -v 3

#-------------------------------------------------------------------------------
# setup all commands used within this flow
TIME_CMD = /usr/bin/time -f 'Elapsed time: %E[h:]min:sec. CPU time: user %U sys %S (%P). Peak memory: %MKB.'
TIME_TEST = $(shell $(TIME_CMD) echo foo 2>/dev/null)
ifeq (, $(strip $(TIME_TEST)))
  TIME_CMD = /usr/bin/time
endif

# The following determine the executable location for each tool used by this flow.
# Priority is given to
#       1 user include path to the executable prior to running make
#       2 user explicit set with variable in Makefile or command line
OPENROAD_EXE            ?= $(shell command -v openroad)
ifeq ($(OPENROAD_EXE),)
#   OPENROAD_EXE           = $(abspath $(FLOW_HOME)/../tools/install/OpenROAD/bin/openroad)
  $(error OpenROAD not found in PATH)
endif
OPENROAD_ARGS            = -no_init $(OR_ARGS)
OPENROAD_CMD             = $(OPENROAD_EXE) -exit $(OPENROAD_ARGS)
OPENROAD_NO_EXIT_CMD     = $(OPENROAD_EXE) $(OPENROAD_ARGS)
OPENROAD_GUI_CMD         = $(OPENROAD_EXE) -gui $(OR_ARGS)

YOSYS_CMD               ?= $(shell command -v yosys)
ifeq ($(YOSYS_CMD),)
#  YOSYS_CMD              = $(abspath $(FLOW_HOME)/../tools/install/yosys/bin/yosys)
  $(error Yosys not found in PATH)
endif

KLAYOUT_CMD             ?= $(shell command -v klayout)

KLAYOUT_FOUND            = $(if $(KLAYOUT_CMD),,$(error KLayout not found in PATH))

#-------------------------------------------------------------------------------
WRAPPED_LEFS = $(foreach lef,$(notdir $(WRAP_LEFS)),$(OBJECTS_DIR)/lef/$(lef:.lef=_mod.lef))
WRAPPED_LIBS = $(foreach lib,$(notdir $(WRAP_LIBS)),$(OBJECTS_DIR)/$(lib:.lib=_mod.lib))
export ADDITIONAL_LEFS += $(WRAPPED_LEFS) $(WRAP_LEFS)
export LIB_FILES += $(WRAP_LIBS) $(WRAPPED_LIBS)

export DONT_USE_LIBS   = $(patsubst %.lib.gz, %.lib, $(addprefix $(OBJECTS_DIR)/lib/, $(notdir $(LIB_FILES))))
export DONT_USE_SC_LIB ?= $(firstword $(DONT_USE_LIBS))

# Stream system used for final result (GDS is default): GDS, GSDII, GDS2, OASIS, or OAS
STREAM_SYSTEM ?= GDS
ifneq ($(findstring GDS,$(shell echo $(STREAM_SYSTEM) | tr '[:lower:]' '[:upper:]')),)
	export STREAM_SYSTEM_EXT := gds
	GDSOAS_FILES = $(GDS_FILES)
	ADDITIONAL_GDSOAS = $(ADDITIONAL_GDS)
	SEAL_GDSOAS = $(SEAL_GDS)
else
	export STREAM_SYSTEM_EXT := oas
	GDSOAS_FILES = $(OAS_FILES)
	ADDITIONAL_GDSOAS = $(ADDITIONAL_OAS)
	SEAL_GDSOAS = $(SEAL_OAS)
endif
export WRAPPED_GDSOAS = $(foreach lef,$(notdir $(WRAP_LEFS)),$(OBJECTS_DIR)/$(lef:.lef=_mod.$(STREAM_SYSTEM_EXT)))


# Targets to harden Blocks in case of hierarchical flow is triggered
build_macros: $(BLOCK_LEFS)

$(foreach block,$(BLOCKS),$(eval ./results/$(PLATFORM)/$(DESIGN_NICKNAME)_$(block)/$(FLOW_VARIANT)/${block}.lef: ./designs/$(PLATFORM)/$(DESIGN_NICKNAME)/${block}/config.mk))
$(foreach block,$(BLOCKS),$(eval ./results/$(PLATFORM)/$(DESIGN_NICKNAME)_$(block)/$(FLOW_VARIANT)/6_final.gds: ./results/$(PLATFORM)/$(DESIGN_NICKNAME)_$(block)/$(FLOW_VARIANT)/${block}.lef))

$(BLOCK_LEFS):
	$(MAKE) DESIGN_CONFIG=$< generate_abstract

# Utility to print tool version information
#-------------------------------------------------------------------------------
.PHONY: versions.txt
versions.txt:
	@$(YOSYS_CMD) -V > $@
	@echo openroad `$(OPENROAD_EXE) -version` >> $@
	@$(KLAYOUT_CMD) -zz -v >> $@

# LVS specific global var for this generator
#-------------------------------------------------------------------------------
#export __open_generator_name__=temp-sense-gen

# Pre-process libraries
# ==============================================================================

# Create temporary Liberty files which have the proper dont_use properties set
# For use with Yosys and ABC
.SECONDEXPANSION:
$(DONT_USE_LIBS): $$(filter %$$(@F) %$$(@F).gz,$(LIB_FILES))
	@mkdir -p $(OBJECTS_DIR)/lib
	$(UTILS_DIR)/markDontUse.py -p "$(DONT_USE_CELLS)" -i $^ -o $@

$(OBJECTS_DIR)/lib/merged.lib:
	$(UTILS_DIR)/mergeLib.pl $(PLATFORM)_merged $(DONT_USE_LIBS) > $@

# Pre-process KLayout tech
# ==============================================================================
$(OBJECTS_DIR)/klayout_tech.lef: $(TECH_LEF)
	@mkdir -p $(OBJECTS_DIR)
	sed '/OR_DEFAULT/d' $< > $@

 $(OBJECTS_DIR)/klayout.lyt: $(KLAYOUT_TECH_FILE) $(OBJECTS_DIR)/klayout_tech.lef
	sed 's,<lef-files>.*</lef-files>,$(foreach file, $(OBJECTS_DIR)/klayout_tech.lef $(SC_LEF) $(ADDITIONAL_LEFS),<lef-files>$(abspath $(file))</lef-files>),g' $< > $@

$(OBJECTS_DIR)/klayout_wrap.lyt: $(KLAYOUT_TECH_FILE) $(OBJECTS_DIR)/klayout_tech.lef
	sed 's,<lef-files>.*</lef-files>,$(foreach file, $(OBJECTS_DIR)/klayout_tech.lef $(WRAP_LEFS),<lef-files>$(abspath $(file))</lef-files>),g' $< > $@
# Create Macro wrappers (if necessary)
# ==============================================================================
WRAP_CFG = $(PLATFORM_DIR)/wrapper.cfg


export TCLLIBPATH := util/cell-veneer $(TCLLIBPATH)
$(WRAPPED_LEFS):
	mkdir -p $(OBJECTS_DIR)/lef $(OBJECTS_DIR)/def
	util/cell-veneer/wrap.tcl -cfg $dWRAP_CFG) -macro $(filter %$(notdir $(@:_mod.lef=.lef)),$(WRAP_LEFS))
	mv $(notdir $@) $@
	mv $(notdir $(@:lef=def)) $(dir $@)../def/$(notdir $(@:lef=def))

$(WRAPPED_LIBS):
	mkdir -p $(OBJECTS_DIR)/lib
	sed 's/library(\(.*\))/library(\1_mod)/g' $(filter %$(notdir $(@:_mod.lib=.lib)),$(WRAP_LIBS)) | sed 's/cell(\(.*\))/cell(\1_mod)/g' > $@

# ==============================================================================
#  ______   ___   _ _____ _   _ _____ ____ ___ ____
# / ___\ \ / / \ | |_   _| | | | ____/ ___|_ _/ ___|
# \___ \\ V /|  \| | | | | |_| |  _| \___ \| |\___ \
#  ___) || | | |\  | | | |  _  | |___ ___) | | ___) |
# |____/ |_| |_| \_| |_| |_| |_|_____|____/___|____/
#
synth: versions.txt \
       $(RESULTS_DIR)/1_synth.v \
       $(RESULTS_DIR)/1_synth.sdc
# ==============================================================================


# Run Synthesis using yosys
#-------------------------------------------------------------------------------
SYNTH_SCRIPT ?= scripts/synth.tcl

$(RESULTS_DIR)/1_1_yosys.v: $(DONT_USE_LIBS) $(WRAPPED_LIBS)
	mkdir -p $(RESULTS_DIR) $(LOG_DIR) $(REPORTS_DIR)
	($(TIME_CMD) yosys -c $(SYNTH_SCRIPT)) 2>&1 | tee $(LOG_DIR)/1_1_yosys.log

$(RESULTS_DIR)/1_synth.v: $(RESULTS_DIR)/1_1_yosys.v
	mkdir -p $(RESULTS_DIR) $(LOG_DIR) $(REPORTS_DIR)
	cp $< $@

$(RESULTS_DIR)/1_synth.sdc: $(SDC_FILE)
	mkdir -p $(RESULTS_DIR) $(LOG_DIR) $(REPORTS_DIR)
	cp $< $@

clean_synth:
	rm -f  $(RESULTS_DIR)/1_*.v $(RESULTS_DIR)/1_synth.sdc
	rm -f  $(REPORTS_DIR)/synth_*
	rm -f  $(LOG_DIR)/1_*
	rm -rf _tmp_yosys-abc-*


# ==============================================================================
#  _____ _     ___   ___  ____  ____  _        _    _   _
# |  ___| |   / _ \ / _ \|  _ \|  _ \| |      / \  | \ | |
# | |_  | |  | | | | | | | |_) | |_) | |     / _ \ |  \| |
# |  _| | |__| |_| | |_| |  _ <|  __/| |___ / ___ \| |\  |
# |_|   |_____\___/ \___/|_| \_\_|   |_____/_/   \_\_| \_|
#
floorplan: $(RESULTS_DIR)/2_floorplan.odb \
           $(RESULTS_DIR)/2_floorplan.sdc
# ==============================================================================

ifneq ($(FOOTPRINT),)
IS_CHIP = 1
else ifneq ($(FOOTPRINT_TCL),)
IS_CHIP = 1
endif

# STEP 1: Translate verilog to odb
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/2_1_floorplan.odb: $(RESULTS_DIR)/1_synth.v $(RESULTS_DIR)/1_synth.sdc $(TECH_LEF) $(SC_LEF) $(ADDITIONAL_LEFS) $(FOOTPRINT) $(SIG_MAP_FILE) $(FOOTPRINT_TCL)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/floorplan.tcl -metrics $(LOG_DIR)/2_1_floorplan.json) 2>&1 | tee $(LOG_DIR)/2_1_floorplan.log


# STEP 2: IO Placement (random)
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/2_2_floorplan_io.odb: $(RESULTS_DIR)/2_1_floorplan.odb $(IO_CONSTRAINTS)
ifndef IS_CHIP
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/io_placement_random.tcl -metrics $(LOG_DIR)/2_2_floorplan_io.json) 2>&1 | tee $(LOG_DIR)/2_2_floorplan_io.log
else
	cp $< $@
endif

# STEP 3: Timing Driven Mixed Sized Placement
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/2_3_floorplan_tdms.odb: $(RESULTS_DIR)/2_2_floorplan_io.odb $(RESULTS_DIR)/1_synth.v $(RESULTS_DIR)/1_synth.sdc $(LIB_FILES)
ifeq ($(MACRO_PLACEMENT),)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/tdms_place.tcl -metrics $(LOG_DIR)/2_3_tdms.json) 2>&1 | tee $(LOG_DIR)/2_3_tdms_place.log
else
	$(info [INFO][FLOW] Using manual macro placement file $(MACRO_PLACEMENT))
	cp $< $@
endif

# STEP 4: Macro Placement
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/2_4_floorplan_macro.odb: $(RESULTS_DIR)/2_3_floorplan_tdms.odb $(RESULTS_DIR)/1_synth.v $(RESULTS_DIR)/1_synth.sdc $(MACRO_PLACEMENT)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/macro_place.tcl -metrics $(LOG_DIR)/2_4_mplace.json) 2>&1 | tee $(LOG_DIR)/2_4_mplace.log

# STEP 5: Tapcell and Welltie insertion
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/2_5_floorplan_tapcell.odb: $(RESULTS_DIR)/2_4_floorplan_macro.odb $(TAPCELL_TCL)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/tapcell.tcl -metrics $(LOG_DIR)/2_5_tapcell.json) 2>&1 | tee $(LOG_DIR)/2_5_tapcell.log

# STEP 6: PDN generation
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/2_6_floorplan_pdn.odb: $(RESULTS_DIR)/2_5_floorplan_tapcell.odb $(PDN_CFG) $(PDN_TCL)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/pdn.tcl -metrics $(LOG_DIR)/2_6_pdn.json) 2>&1 | tee $(LOG_DIR)/2_6_pdn.log

$(RESULTS_DIR)/2_floorplan.odb: $(RESULTS_DIR)/2_6_floorplan_pdn.odb
	cp $< $@

$(RESULTS_DIR)/2_floorplan.sdc: $(RESULTS_DIR)/2_1_floorplan.odb

clean_floorplan:
	rm -f $(RESULTS_DIR)/2_*floorplan*.odb $(RESULTS_DIR)/2_floorplan.sdc $(RESULTS_DIR)/2_floorplan.v $(RESULTS_DIR)/2_*floorplan.def
	rm -f $(REPORTS_DIR)/2_*
	rm -f $(LOG_DIR)/2_*

# ==============================================================================
#  ____  _        _    ____ _____
# |  _ \| |      / \  / ___| ____|
# | |_) | |     / _ \| |   |  _|
# |  __/| |___ / ___ \ |___| |___
# |_|   |_____/_/   \_\____|_____|
#
place: $(RESULTS_DIR)/3_place.odb \
       $(RESULTS_DIR)/3_place.sdc

# STEP 0.0: Temporarily convert back to .DEF to run the RO placement, then revert

$(RESULTS_DIR)/2_floorplan.def: $(RESULTS_DIR)/2_floorplan.odb $(RESULTS_DIR)/2_floorplan.sdc
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/odb_to_def.tcl -metrics $(LOG_DIR)/3_0_conversion.json) 2>&1 | tee $(LOG_DIR)/3_0_conversion.log

# STEP 0.3: Place RO to fixed coordinates. Variables are defined inside of config.mk in flow

$(RESULTS_DIR)/2_floorplan_ro.def: $(RESULTS_DIR)/2_floorplan.def
	python3 util/place_ro.py --inputDef $(RESULTS_DIR)/2_floorplan.def --outputDef $(RESULTS_DIR)/2_floorplan_ro.def --coreDim $(RO_CORE_DIM) --arrayDim $(RO_ARRAY_DIM) --cellDim $(RO_CELL_DIM) --targetInst cryo_ro_1 --coreDieOffset $(RO_OFFSET)

# STEP 0.6: Revert .DEF back to .ODB to finish p&r

$(RESULTS_DIR)/2_floorplan_ro.odb: $(RESULTS_DIR)/2_floorplan_ro.def
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/def_to_odb.tcl -metrics $(LOG_DIR)/3_1_conversion.json) 2>&1 | tee $(LOG_DIR)/3_1_conversion.log

# ==============================================================================
# STEP 1: Global placement without placed IOs, timing-driven, and routability-driven.
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/3_1_place_gp.odb: $(RESULTS_DIR)/2_floorplan_ro.odb $(RESULTS_DIR)/2_floorplan.sdc $(LIB_FILES)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/global_place.tcl -metrics $(LOG_DIR)/3_1_place_gp.json) 2>&1 | tee $(LOG_DIR)/3_1_place_gp.log

# STEP 2: IO placement (non-random)
#-------------------------------------------------------------------------------
#$(RESULTS_DIR)/3_2_place_iop.odb: $(RESULTS_DIR)/3_1_place_gp_skip_io.odb $(IO_CONSTRAINTS)
#ifndef IS_CHIP
#	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/io_placement.tcl -metrics $(LOG_DIR)/3_2_place_iop.json) 2>&1 | tee $(LOG_DIR)/3_2_place_iop.log
#else
#	cp $< $@
#endif

# STEP 3: Global placement with placed IOs, timing-driven, and routability-driven.
#-------------------------------------------------------------------------------
#$(RESULTS_DIR)/3_3_place_gp.odb: $(RESULTS_DIR)/3_2_place_iop.odb $(RESULTS_DIR)/2_floorplan.sdc $(LIB_FILES)
#	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/global_place.tcl -metrics $(LOG_DIR)/3_3_place_gp.json) 2>&1 | tee $(LOG_DIR)/3_3_place_gp.log

# # STEP 4: Resizing & Buffering (not needed for the tempsense)
# #-------------------------------------------------------------------------------
# $(RESULTS_DIR)/3_4_place_resized.odb: $(RESULTS_DIR)/3_3_place_gp.odb $(RESULTS_DIR)/2_floorplan.sdc
# 	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/resize.tcl -metrics $(LOG_DIR)/3_4_resizer.json) 2>&1 | tee $(LOG_DIR)/3_4_resizer.log
#
# clean_resize:
# 	rm -f $(RESULTS_DIR)/3_4_place_resized.odb

#$(RESULTS_DIR)/3_4_place_resized.odb: $(RESULTS_DIR)/3_2_place_iop.odb # just copy file to not break dependencies
#	cp $< $@

# STEP 5: Detail placement
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/3_5_place_dp.odb: $(RESULTS_DIR)/3_1_place_gp.odb $(RESULTS_DIR)/2_floorplan.sdc $(LIB_FILES)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/detail_place.tcl -metrics $(LOG_DIR)/3_5_opendp.json) 2>&1 | tee $(LOG_DIR)/3_5_opendp.log

$(RESULTS_DIR)/3_place.odb: $(RESULTS_DIR)/3_5_place_dp.odb
	cp $< $@

$(RESULTS_DIR)/3_place.sdc: $(RESULTS_DIR)/2_floorplan.sdc
	cp $< $@


# Clean Targets
#-------------------------------------------------------------------------------
clean_place:
	rm -f $(RESULTS_DIR)/3_*place*.odb
	rm -f $(RESULTS_DIR)/3_place.sdc
	rm -f $(REPORTS_DIR)/3_*
	rm -f $(LOG_DIR)/3_*

# ==============================================================================
#   ____ _____ ____
#  / ___|_   _/ ___|
# | |     | | \___ \
# | |___  | |  ___) |
#  \____| |_| |____/
#
cts: $(RESULTS_DIR)/4_cts.odb \
     $(RESULTS_DIR)/4_cts.sdc
# ==============================================================================

# # Run TritonCTS (not needed for the tempsense)
# # ------------------------------------------------------------------------------
# $(RESULTS_DIR)/4_1_cts.odb: $(RESULTS_DIR)/3_place.odb $(RESULTS_DIR)/3_place.sdc
# 	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/cts.tcl -metrics $(LOG_DIR)/4_1_cts.json) 2>&1 | tee $(LOG_DIR)/4_1_cts.log
$(RESULTS_DIR)/4_1_cts.odb: $(RESULTS_DIR)/3_place.odb $(RESULTS_DIR)/3_place.sdc # just copy file to not break dependencies
	cp $< $@

# Filler cell insertion
# ------------------------------------------------------------------------------
$(RESULTS_DIR)/4_2_cts_fillcell.odb: $(RESULTS_DIR)/4_1_cts.odb
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/fillcell.tcl -metrics $(LOG_DIR)/4_2_cts_fillcell.json) 2>&1 | tee $(LOG_DIR)/4_2_cts_fillcell.log

# $(RESULTS_DIR)/4_cts.sdc: $(RESULTS_DIR)/4_cts.odb
# Changed due to suppresion of TritonCTS step
$(RESULTS_DIR)/4_cts.sdc: $(RESULTS_DIR)/3_place.sdc # just copy file to not break dependencies
	cp $< $@

$(RESULTS_DIR)/4_cts.odb: $(RESULTS_DIR)/4_2_cts_fillcell.odb
	cp $< $@

clean_cts:
	rm -rf $(RESULTS_DIR)/4_*cts*.odb $(RESULTS_DIR)/4_cts.sdc $(RESULTS_DIR)/4_cts.v $(RESULTS_DIR)/4_*cts.def
	rm -f  $(REPORTS_DIR)/4_*
	rm -f  $(LOG_DIR)/4_*


# ==============================================================================
#  ____   ___  _   _ _____ ___ _   _  ____
# |  _ \ / _ \| | | |_   _|_ _| \ | |/ ___|
# | |_) | | | | | | | | |  | ||  \| | |  _
# |  _ <| |_| | |_| | | |  | || |\  | |_| |
# |_| \_\\___/ \___/  |_| |___|_| \_|\____|
#
route: $(RESULTS_DIR)/5_route.odb \
       $(RESULTS_DIR)/5_route.sdc
# ==============================================================================


# STEP 1: Run global route
#-------------------------------------------------------------------------------
$(RESULTS_DIR)/5_1_grt.odb: $(RESULTS_DIR)/4_cts.odb $(RESULTS_DIR)/4_cts.sdc $(FASTROUTE_TCL) $(PRE_GLOBAL_ROUTE)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/global_route.tcl -metrics $(LOG_DIR)/5_1_fastroute.json) 2>&1 | tee $(LOG_DIR)/5_1_fastroute.log


# STEP 2: Run detailed route
#-------------------------------------------------------------------------------
ifeq ($(USE_WXL),)
$(RESULTS_DIR)/5_2_route.odb: $(RESULTS_DIR)/5_1_grt.odb
else
$(RESULTS_DIR)/5_2_route.odb: $(RESULTS_DIR)/4_cts.odb
endif
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/detail_route.tcl -metrics $(LOG_DIR)/5_2_TritonRoute.json) 2>&1 | tee $(LOG_DIR)/5_2_TritonRoute.log

$(RESULTS_DIR)/5_route.odb: $(RESULTS_DIR)/5_2_route.odb
	cp $< $@

$(RESULTS_DIR)/5_route.sdc: $(RESULTS_DIR)/4_cts.sdc
	cp $< $@

clean_route:
	rm -rf output*/ results*.out.dmp layer_*.mps
	rm -rf *.gdid *.log *.met *.sav *.res.dmp
	rm -rf $(RESULTS_DIR)/route.guide $(RESULTS_DIR)/output_guide.mod $(RESULTS_DIR)/updated_clks.sdc
	rm -rf $(RESULTS_DIR)/5_*.odb $(RESULTS_DIR)/5_route.sdc
	rm -f  $(REPORTS_DIR)/5_*
	rm -f  $(LOG_DIR)/5_*

klayout_tr_rpt: $(RESULTS_DIR)/5_route.def $(OBJECTS_DIR)/klayout.lyt
	$(call KLAYOUT_FOUND)
	$(KLAYOUT_CMD) -rd in_drc="$(REPORTS_DIR)/5_route_drc.rpt" \
	        -rd in_def="$<" \
	        -rd tech_file=$(OBJECTS_DIR)/klayout.lyt \
	        -rm $(UTILS_DIR)/viewDrc.py

klayout_guides: $(RESULTS_DIR)/5_route.def $(OBJECTS_DIR)/klayout.lyt
	$(call KLAYOUT_FOUND)
	$(KLAYOUT_CMD) -rd in_guide="$(RESULTS_DIR)/route.guide" \
	        -rd in_def="$<" \
	        -rd net_name=$(GUIDE_NET) \
	        -rd tech_file=$(OBJECTS_DIR)/klayout.lyt \
	        -rm $(UTILS_DIR)/viewGuide.py

# ==============================================================================
#  _____ ___ _   _ ___ ____  _   _ ___ _   _  ____
# |  ___|_ _| \ | |_ _/ ___|| | | |_ _| \ | |/ ___|
# | |_   | ||  \| || |\___ \| |_| || ||  \| | |  _
# |  _|  | || |\  || | ___) |  _  || || |\  | |_| |
# |_|   |___|_| \_|___|____/|_| |_|___|_| \_|\____|
#
GDS_FINAL_FILE = $(RESULTS_DIR)/6_final.$(STREAM_SYSTEM_EXT)
finish: $(LOG_DIR)/6_report.log \
        $(RESULTS_DIR)/6_final.v \
        $(RESULTS_DIR)/6_final.sdc \
        $(GDS_FINAL_FILE) 
# ==============================================================================

ifneq ($(USE_FILL),)
$(RESULTS_DIR)/6_1_fill.odb: $(RESULTS_DIR)/5_route.odb $(FILL_CONFIG)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/density_fill.tcl -metrics $(LOG_DIR)/6_density_fill.json) 2>&1 | tee $(LOG_DIR)/6_density_fill.log
else
$(RESULTS_DIR)/6_1_fill.odb: $(RESULTS_DIR)/5_route.odb
	cp $< $@
endif

$(RESULTS_DIR)/6_1_fill.sdc: $(RESULTS_DIR)/5_route.sdc
	cp $< $@

$(RESULTS_DIR)/6_final.sdc: $(RESULTS_DIR)/5_route.sdc
	cp $< $@

$(LOG_DIR)/6_report.log: $(RESULTS_DIR)/6_1_fill.odb $(RESULTS_DIR)/6_1_fill.sdc
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/final_report.tcl -metrics $(LOG_DIR)/6_report.json) 2>&1 | tee $(LOG_DIR)/6_report.log

$(RESULTS_DIR)/6_final.def: $(LOG_DIR)/6_report.log

generate_abstract: $(RESULTS_DIR)/6_final.gds $(RESULTS_DIR)/6_final.def  $(RESULTS_DIR)/6_final.v
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/generate_abstract.tcl -metrics $(LOG_DIR)/generate_abstract.json) 2>&1 | tee $(LOG_DIR)/generate_abstract.log

# Merge wrapped macros using Klayout
#-------------------------------------------------------------------------------
$(WRAPPED_GDSOAS): $(OBJECTS_DIR)/klayout_wrap.lyt $(WRAPPED_LEFS)
	$(call KLAYOUT_FOUND)
	($(TIME_CMD) $(KLAYOUT_CMD) -zz -rd design_name=$(basename $(notdir $@)) \
	        -rd in_def=$(OBJECTS_DIR)/def/$(notdir $(@:$(STREAM_SYSTEM_EXT)=def)) \
	        -rd in_files="$(ADDITIONAL_GDSOAS)" \
	        -rd config_file=$(FILL_CONFIG) \
	        -rd seal_file="" \
	        -rd out_file=$@ \
	        -rd tech_file=$(OBJECTS_DIR)/klayout_wrap.lyt \
	        -rd layer_map=$(GDS_LAYER_MAP) \
	        -r $(UTILS_DIR)/def2stream.py) 2>&1 | tee $(LOG_DIR)/6_merge_$(basename $(notdir $@)).log

# Merge GDS using Klayout
#-------------------------------------------------------------------------------
GDS_MERGED_FILE = $(RESULTS_DIR)/6_1_merged.$(STREAM_SYSTEM_EXT)
$(GDS_MERGED_FILE): $(RESULTS_DIR)/6_final.def $(OBJECTS_DIR)/klayout.lyt $(GDSOAS_FILES) $(WRAPPED_GDSOAS) $(SEAL_GDSOAS)
	$(call KLAYOUT_FOUND)
	($(TIME_CMD) $(KLAYOUT_CMD) -zz -rd design_name=$(DESIGN_NAME) \
	        -rd in_def=$< \
	        -rd in_files="$(GDSOAS_FILES) $(WRAPPED_GDSOAS)" \
	        -rd config_file=$(FILL_CONFIG) \
	        -rd seal_file="$(SEAL_GDSOAS)" \
	        -rd out_file=$@ \
	        -rd tech_file=$(OBJECTS_DIR)/klayout.lyt \
	        -rd layer_map=$(GDS_LAYER_MAP) \
	        -r $(UTILS_DIR)/def2stream.py) 2>&1 | tee $(LOG_DIR)/6_1_merge.log

$(RESULTS_DIR)/6_final.v: $(LOG_DIR)/6_report.log

$(GDS_FINAL_FILE): $(GDS_MERGED_FILE)
	cp $^ $@

drc: $(REPORTS_DIR)/6_drc.lyrdb

$(REPORTS_DIR)/6_drc.lyrdb: $(GDS_FINAL_FILE) $(KLAYOUT_DRC_FILE)
ifneq ($(KLAYOUT_DRC_FILE),)
	$(call KLAYOUT_FOUND)
	($(TIME_CMD) $(KLAYOUT_CMD) -zz -rd in_gds="$<" \
	        -rd report_file=$(abspath $@) \
	        -r $(KLAYOUT_DRC_FILE)) 2>&1 | tee $(LOG_DIR)/6_drc.log
	# Hacky way of getting DRV count (don't error on no matches)
	grep -c "<value>" $@ > $(REPORTS_DIR)/6_drc_count.rpt || [[ $$? == 1 ]]
else
	echo "DRC not supported on this platform" > $@
endif

$(RESULTS_DIR)/6_final.cdl: $(RESULTS_DIR)/6_final.v
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/cdl.tcl) 2>&1 | tee $(LOG_DIR)/6_cdl.log

$(OBJECTS_DIR)/6_final_concat.cdl: $(RESULTS_DIR)/6_final.cdl $(CDL_FILE)
	cat $^ > $@

lvs: $(RESULTS_DIR)/6_lvs.lvsdb

$(RESULTS_DIR)/6_lvs.lvsdb: $(GDS_FINAL_FILE) $(KLAYOUT_LVS_FILE) $(OBJECTS_DIR)/6_final_concat.cdl
ifneq ($(KLAYOUT_LVS_FILE),$(GDS_FINAL_FILE))
	echo $(call KLAYOUT_FOUND)
	($(TIME_CMD) $(KLAYOUT_CMD) -b -rd in_gds="$<" \
	        -rd cdl_file=$(abspath $(OBJECTS_DIR)/6_final_concat.cdl) \
	        -rd report_file=$(abspath $@) \
	        -r $(KLAYOUT_LVS_FILE)) 2>&1 | tee $(LOG_DIR)/6_lvs.log
else
	echo "LVS not supported on this platform" > $@
endif

# Other tools for DRC and LVS
export COMMON_VERIF_DIR = $(COMMON_HOME)/drc-lvs-check

magic_drc: $(RESULTS_DIR)/6_final.gds
ifeq ($(shell command -v magic),)
	$(error Magic not found in PATH)
endif
	$(COMMON_VERIF_DIR)/run_drc.sh
	 magic -rcfile $(COMMON_VERIF_DIR)/sky130A/sky130A.magicrc -noconsole -dnull $(COMMON_VERIF_DIR)/magic_commands.tcl < /dev/null

netgen_lvs: $(RESULTS_DIR)/6_final.gds $(RESULTS_DIR)/6_final.cdl
ifeq ($(shell command -v netgen),)
	$(error Netgen not found in PATH)
endif
ifeq ($(shell command -v magic),)
	$(error Magic not found in PATH)
endif
	mkdir -p $(OBJECTS_DIR)/netgen_lvs/spice $(OBJECTS_DIR)/netgen_lvs/ext
	python $(UTILS_DIR)/openfasoc/cdl_parser.py -i $(RESULTS_DIR)/6_final.cdl -s $(PLATFORM_DIR)/cdl/$(SPICE_FILE) -o $(OBJECTS_DIR)/netgen_lvs/spice/$(DESIGN_NAME).spice
	$(COMMON_VERIF_DIR)/run_lvspex.sh $(RESULTS_DIR)/6_final.gds $(DESIGN_NAME) $(REPORTS_DIR)/6_final_lvs.rpt

clean_finish:
	rm -rf $(RESULTS_DIR)/6_*.gds $(RESULTS_DIR)/6_*.oas $(RESULTS_DIR)/6_*.odb $(RESULTS_DIR)/6_*.v $(RESULTS_DIR)/6_*.sdc $(RESULTS_DIR)/6_*.spef
	rm -rf $(REPORTS_DIR)/6_*.rpt
	rm -f  $(LOG_DIR)/6_*



# ==============================================================================
#  __  __ ___ ____   ____
# |  \/  |_ _/ ___| / ___|
# | |\/| || |\___ \| |
# | |  | || | ___) | |___
# |_|  |_|___|____/ \____|
#
# ==============================================================================

all: $(SDC_FILE) $(WRAPPED_LIBS) $(DONT_USE_LIBS) $(OBJECTS_DIR)/TritonRoute.param $(OBJECTS_DIR)/klayout.lyt $(WRAPPED_GDS)
	mkdir -p $(RESULTS_DIR) $(LOG_DIR) $(REPORTS_DIR)
	($(TIME_CMD) $(OPENROAD_CMD) $(SCRIPTS_DIR)/run_all.tcl -metrics $(LOG_DIR)/run_all.json) 2>&1 | tee $(LOG_DIR)/run_all.log

clean:
	@echo
	@echo "Make clean disabled."
	@echo "Use make clean_all or clean individual steps:"
	@echo "  clean_synth clean_floorplan clean_place clean_cts clean_route clean_finish"
	@echo

clean_all: clean_synth clean_floorplan clean_place clean_cts clean_route clean_finish
	rm -rf $(OBJECTS_DIR)
	rm -f cryoInst*

nuke: clean_test clean_issues
	rm -rf ./results ./logs ./reports ./objects
	rm -rf layer_*.mps macrocell.list *best.plt *_pdn.def dummy.guide run.param
	rm -rf *.rpt *.rpt.old *.def.v pin_dumper.log
	rm -rf versions.txt
	rm -f *.ext *.spice


# DEF/GDS viewer shortcuts
#-------------------------------------------------------------------------------
RESULTS_DEF = $(notdir $(sort $(wildcard $(RESULTS_DIR)/*.def)))
RESULTS_GDS = $(notdir $(sort $(wildcard $(RESULTS_DIR)/*.gds)))
$(foreach file,$(RESULTS_DEF) $(RESULTS_GDS),klayout_$(file)): klayout_%: $(OBJECTS_DIR)/klayout.lyt
	klayout -nn $(OBJECTS_DIR)/klayout.lyt $(RESULTS_DIR)/$*

gui_final: $(RESULTS_DIR)/6_final.def
	openroad -gui $(SCRIPTS_DIR)/gui.tcl


# Utilities
#-------------------------------------------------------------------------------
include $(UTILS_DIR)/utils.mk
export PRIVATE_DIR = ../../private_tool_scripts
-include $(PRIVATE_DIR)/util/utils.mk
