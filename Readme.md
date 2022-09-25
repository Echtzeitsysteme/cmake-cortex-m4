# Rennorbs experiments to port the winidea setup for compiling to cortex-m4 to a cross platform cmake project

## **DOES NOT FULLY WORK IN ITS CURRENT STATE**
current issues: 
- [lib/gfx.c](lib/gfx.c:539) does not properly link against sprintf because of missing __aeabi_f2d. It's not the 'float literals are doubles by default' issue, its something else. Enable [-Wdouble-promotion](./CMakeLists.txt:94) to see that it's a different issue. 
  The documentation for my compiler (`share/doc/gcc-arm-none-eabi/html/libc/siprintf.html`) specifies that only integer format specifiers are supported by this version.
  However, seeing as this function isn't currently used currently used (in the main project) it should just be save to comment out the call to sprintf.
- flashing target. ... yea, kindof important part 

## Requirements
Compilation: 
- [gcc-arm-none-eabi](https://developer.arm.com/downloads/-/gnu-rm)
- [cmake](https://cmake.org/download/)
- some generator, could be msvc, [Ninja](https://github.com/ninja-build/ninja/releases) or something else (tested with Ninja)
- an "installation" of the [PDL (PeripheralDriverLibrary)](http://www.cypress.com/documentation/software-and-drivers/peripheral-driver-library-pdl) from Cypress 

Flashing/Debugging: (WIP)
-  CmsisDapDriver (CmsisDapDriverInstallerV14.exe), might also work with different versions.
- [DapLink](https://daplink.io/) might also be an option
## Instructions

0. clone the project: `git clone git@github.com:SaculRennorb/cmake-cortex-m4.git`
1. move to the project root: `cd cmake-cortex-m4`
2. configure (this specific command requires ninja): `cmake -G Ninja -B build` . add `-DBOARD=<BOARD>` and/or `-DCONFIGURATION=<release|debug>` to configure those parameters
4. build: `cmake --build build --target <BOARD>_<CONFIG>` outputs will be in `build/bin`
5. flash (TODO)