# Generating DSA Files

After completing your hardware platform design and generating a valid bitstream using the Vivado Design Suite, you are ready to create a Device Support Archive (DSA) file for use with the SDAccel Development Environment. A DSA is a single-file capture of the hardware platform, to be handed off by the platform developer for use with the SDAccel Environment.

_**These properties are not saved in the Vivado project so they must be set**_
_**each time a DSA is created. You can create a TCL script to run prior to creating the DSA.**_

The following three properties are required by the write_dsa Tcl command, and must be defined
prior to generating a DSA. If these properties are not found, the write_dsa command will issue an
error and stop. 
The values shown are example values.
***
> 1. set_property dsa.vendor "xilinx" [current_project]
> 2. set_property dsa.name "1ddr" [current_project]
> 3. set_property dsa.boardId "adm-board" [current_project]
***
The following properties have default values 
but can be defined to specify the DSA file version, 
enable the partial reconfiguration (PR) flow, 
or 
capture the synthesis checkpoint within the DSA.
***
Default value is 0.0

```tcl 
set_property dsa.version "1.0" [current_project]
set_property dsa.uses_pr true [current_project]
set_property dsa.static_synth_checkpoint false [current_project]
```
***
The following properties can be used to set the PCIe Id and Board attributes for the Board section if a
board file is not yet available.
> 
```tcl 
set_property dsa.pcie_id_vendor "0x10ee" [current_project]
set_property dsa.pcie_id_device "0x8038" [current_project]
set_property dsa.pcie_id_subsystem "0x0011" [current_project]
set_property dsa.board_name "alpha-data.com:adm-pcie3-ku3:1.0" [current_project]
set_property dsa.board_interface_type "gen3x8" [current_project]
set_property dsa.board_memories {{ddr3 8GB} {ddr4 16GB} [current_project]
set_property dsa.board_interface_name "PCIe" [current_project]
set_property dsa.board_vendor "alpha-data.com" [current_project]
```
***

# Get current directory, used throughout script
```tcl 
set launchDir [file dirname [file normalize [info script]]]
set sourcesDir ${launchDir}/sources
```
# Create the project using the board local repo
```tcl 
set projName "xcl_design"
set projPart "xcku115-flvb2104-2-e"
set projBoardPart "xilinx.com:xil-accel-rd-ku115:part0:1.0"
set_param board.RepoPaths ${sourcesDir}/boardrepo/xil-accel-rd-ku115/1.0
get_board_parts
create_project $projName ./$projName -part $projPart
set_property board_part $projBoardPart [current_project]

# Create the BD
```tcl 
create_bd_design xcl_design
```
# Set required environment variables and params
```tcl 
set ::env(OCL_BLOCK_ADVANCED) 1
set ::env(XIL_IFX_SC_PIPELINE_EXPERIMENT_LEVEL) 0
set_param dsa.expandedPRRegion 1
set_param synth.elaboration.rodinMoreOptions "rt::set_parameter advancedConstPropAcrossHier true"
set_param chipscope.enablePRFlow true

```



# Import HDL, XDC, and other files
```tcl 
import_files -norecurse ${sourcesDir}/hdl/iob_static_qspi.v
import_files -norecurse ${sourcesDir}/hdl/startup_wrapper.v
import_files -fileset constrs_1 -norecurse ${sourcesDir}/constraints/xcl_design.xdc
update_compile_order -fileset sources_1
update_compile_order -fileset sim_1
```
# Set DSA project properties
```tcl 
set_property dsa.vendor               "xilinx"             [current_project]
set_property dsa.board_id             "xil-accel-rd-ku115" [current_project]
set_property dsa.name                 "4ddr-xpr"           [current_project]
set_property dsa.version              "3.2"                [current_project]
set_property dsa.flash_interface_type "spix8"              [current_project]
set_property dsa.flash_offset_address "0x4000000"          [current_project]
set_property dsa.description          "See Notes 1." [current_project]
```

> _**Notes 1:**_ This platform targets the Xilinx Development Board for Acceleration with Kintex UltraScale KU115 FPGA. This high-performance acceleration platform features four channels of DDR4-2400 SDRAM, the expanded partial reconfiguration flow for high fabric resource availability, and Xilinx DMA Subsystem for PCI Express with PCIe Gen3 x8 connectivity

# Set any other project properties
```tcl 
set_property STEPS.OPT_DESIGN.TCL.PRE ${sourcesDir}/misc/xpr_preopt.tcl [get_runs impl_1]
set_property STEPS.OPT_DESIGN.TCL.POST ${sourcesDir}/misc/xpr_postopt.tcl [get_runs impl_1]
set_property STEPS.ROUTE_DESIGN.TCL.POST ${sourcesDir}/misc/xpr_postroute.tcl [get_runs impl_1]
```
# Source the BD Tcl file to construct the BD contents
```tcl 
source ${sourcesDir}/bd/xcl_design.tcl
```
# Regenerate layout, validate, and save the BD
```tcl 
regenerate_bd_layout
validate_bd_design -force
save_bd_design
```

# Write BD wrapper HDL

```tcl 
set_property generate_synth_checkpoint true [get_files xcl_design.bd]
add_files -norecurse [make_wrapper -files [get_files xcl_design.bd] -top]
update_compile_order -fileset sources_1
update_compile_order -fileset sim_1
```

***

1. open_run impl_1
2. write_dsa -include_bit
3. validate_dsa xcl_design.dsa -verbose
***
# VIVADO CONSOLE

```tcl 
write_dsa -include_bit
INFO: [Vivado 12-4895] Creating DSA: xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.dsa ...
Writing placer database...
Writing XDEF routing.
Writing XDEF routing logical nets.
Writing XDEF routing special nets.
Write XDEF Complete: Time (s): cpu = 00:02:49 ; elapsed = 00:00:55 . Memory (MB): peak = 6796.766 ; gain = 459.152
INFO: [Common 17-1381] The checkpoint 'C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.dcp' has been generated.
write_checkpoint: Time (s): cpu = 00:03:42 ; elapsed = 00:01:41 . Memory (MB): peak = 6796.766 ; gain = 459.152
Command: write_cfgmem -force -format mcs -interface spix8 -size 1024 -loadbit {up 0x4000000 C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.bit} -file C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/firmware/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.mcs
Creating config memory files...
Creating bitstream load up from address 0x04000000
Loading bitfile C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.bit
Writing file C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/firmware/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2_primary.mcs
Writing file C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/firmware/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2_secondary.mcs
Writing log file C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/firmware/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2_primary.prm
===================================
Configuration Memory information
===================================
File Format        MCS
Interface          SPIX8
Size               512M
Start Address      0x00000000
End Address        0x1FFFFFFF

Addr1         Addr2         Date                    File(s)
0x02000000    0x02BD2E07    May 19 23:10:56 2017    C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.bit
Writing log file C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/firmware/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2_secondary.prm
===================================
Configuration Memory information
===================================
File Format        MCS
Interface          SPIX8
Size               512M
Start Address      0x00000000
End Address        0x1FFFFFFF

Addr1         Addr2         Date                    File(s)
0x02000000    0x02BD2E07    May 19 23:10:56 2017    C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/.Xil/Vivado-30812-SakinderLaptop1/dsa/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.bit
2 Infos, 0 Warnings, 0 Critical Warnings and 0 Errors encountered.
write_cfgmem completed successfully
write_cfgmem: Time (s): cpu = 00:00:40 ; elapsed = 00:00:38 . Memory (MB): peak = 6796.766 ; gain = 0.000
INFO: [Vivado 12-4896] Successfully created DSA: C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.dsa
write_dsa: Time (s): cpu = 00:04:53 ; elapsed = 00:02:48 . Memory (MB): peak = 6796.766 ; gain = 459.152
C:/RTL/V2016x/P/V64Vx2/vivado/xcl_design/xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.dsa
validate_dsa xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.dsa -verbose
INFO: [Vivado 12-4468] Found XML metadata file: dsa.xml
INFO: [Vivado 12-4464] Found BIT file: xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.bit
INFO: [Vivado 12-4465] Found DCP file: xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2.dcp
INFO: [Vivado 12-4466] Found MCS file in firmware directory: xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2_primary.mcs
INFO: [Vivado 12-4466] Found MCS file in firmware directory: xilinx_xil-accel-rd-ku115_4ddr-xpr_3_2_secondary.mcs
```