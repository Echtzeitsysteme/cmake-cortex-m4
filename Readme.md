# Rennorbs experiments to port the winidea setup for compiling to cortex-m4 to a cross platform cmake project

## **DOES NOT FULLY WORK IN ITS CURRENT STATE**
current issues: 
- [lib/gfx.c](lib/gfx.c:539) does not properly link against sprintf because of missing __aeabi_f2d. It's not the 'float literals are doubles by default' issue, its something else. Enable [-Wdouble-promotion](CMakeLists.txt:94) to see that it's a different issue. 
- flashing target. ... yea, kindof important part 

## Instructions

0. clone the project: `git clone git@github.com:SaculRennorb/cmake-cortex-m4.git`
1. move to the project root: `cd cmake-cortex-m4`
2. configure (this specific command requires ninja): `cmake -G Ninja -B build` . add `-DBOARD=<BOARD>` and/or `-DCONFIGURATION=<release|debug>` to configure those parameters
4. build: `cmake --build build --target <BOARD>_<CONFIG>` outputs will be in `build/bin`