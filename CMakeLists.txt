cmake_minimum_required(VERSION 3.3)

set(PDL "C:/PortableApps/Cypress/PDL/2.1.0")
#set(COMPILER_PATH_HINT "C:/PortableApps/iSYSTEM/winIDEAOpen9/gcc/arm")
set(COMPILER_PATH_HINT "C:/PortableApps/gcc-arm-none-eabi-10.3-2021.10/")

find_program(ARM_NONE_EABI_GCC
  "arm-none-eabi-gcc"
  HINTS "${COMPILER_PATH_HINT}/bin"
  REQUIRED
)
find_program(ARM_NONE_EABI_OBJCOPY
  "arm-none-eabi-objcopy"
  HINTS "${COMPILER_PATH_HINT}/bin"
  REQUIRED
)


set(CMAKE_SYSTEM_NAME      "Generic")
set(CMAKE_SYSTEM_PROCESSOR "cortex-m4")
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
set(CMAKE_CROSSCOMPILING ON)

set(CMAKE_ASM_COMPILER ${ARM_NONE_EABI_GCC})
set(CMAKE_C_COMPILER   ${ARM_NONE_EABI_GCC})

project(board ASM C)

set(CMAKE_ASM_OUTPUT_EXTENSION ".o")
set(CMAKE_C_OUTPUT_EXTENSION   ".o")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "bin")
set(CMAKE_EXECUTABLE_SUFFIX ".elf")

include_directories(
  "lib"
  "solution"
  "${COMPILER_PATH_HINT}/arm-none-eabi/include"
  "${PDL}/devices/fm4/s6e2ccxj/common"
  "${PDL}/cmsis/include"
  "${PDL}/drivers/mfs"
  "${PDL}/drivers/dma"
  "${PDL}/utilities"
  "${PDL}/utilities/printf_scanf"
  "${PDL}/drivers/adc"
  "${PDL}/drivers"
  "."
)

#file(GLOB LIB_FILES "lib/*.c")
set(LIB_FILES 
  "lib/delay.c"
  "lib/init.c"
  #"lib/cppp_interrupts.c" ## collides
  "lib/uart_multicon.c"
  "lib/uart_usb.c"
  "lib/rgb_led.c"
  "lib/analog.c"
  "lib/board_test.c"
  "lib/acceleration_app.c"
  "lib/acceleration_core.c"
  "lib/gfx.c"
  "lib/lcd.c"
)
file(GLOB SRC_FILES "src/*.c")
file(GLOB SOLUTION_FILES "solution/*.c")
set(PDL_FILES 
  "${PDL}/drivers/adc/adc.c"
  "${PDL}/drivers/rtc/rtc.c"
  "${PDL}/drivers/exint/exint.c"
  "${PDL}/drivers/clk/clk.c"
  "${PDL}/drivers/dma/dma.c"
  "${PDL}/drivers/vbat/vbat.c"
  "${PDL}/drivers/mfs/mfs.c"
  "${PDL}/drivers/pdl.c"
  "startup_s6e2cc.s"
  "${PDL}/devices/fm4/s6e2ccxj/common/system_s6e2cc.c"
  "${PDL}/utilities/printf_scanf/uart_io.c"
)

set(SOURCES ${LIB_FILES} ${SRC_FILES} ${SOLUTION_FILES} ${PDL_FILES})

## global
##add_custom_command(TARGET board POST_BUILD 
##  COMMAND arm-none-eabi-objcopy $(CMPDIR) ${CMAKE_RUNTIME_OUTPUT_DIRECTORY} USBDBoot 9.6
##)

string(JOIN "$<SEMICOLON>" COMMON_COMPILE_OPTIONS 
  "-mcpu=${CMAKE_SYSTEM_PROCESSOR}"
  "-x" "c"
  "-mthumb"
  "-g3"
  "-c"
  "-std=c11"
  #"-Wall"
  #"-Wdouble-promotion"
)
string(JOIN "$<SEMICOLON>" COMMON_ASSEMBLE_OPTIONS 
  "-mcpu=${CMAKE_SYSTEM_PROCESSOR}"
  "-mthumb"
  "-g3"
  "-Wdouble-promotion"
)
add_compile_options(
  $<$<COMPILE_LANGUAGE:ASM>:${COMMON_ASSEMBLE_OPTIONS}>
  $<$<COMPILE_LANGUAGE:C>:${COMMON_COMPILE_OPTIONS}>
)

set(COMMON_LINKER_OPTIONS "-n" "--gc-sections")
set(COMMON_COMPILER_LINK_OPTIONS
  "-mthumb"
  "-mcpu=${CMAKE_SYSTEM_PROCESSOR}"
  "-nostartfiles"
  "-nostdlib"
  "-nodefaultlibs"
  "-fno-exceptions"
  "--specs=nano.specs"
  #"-u" "_printf_float" // requires even more aeabi memes
)
set(CMAKE_C_LINKER_WRAPPER_FLAG "-Xlinker" " ")
unset(CMAKE_C_LINKER_WRAPPER_FLAG_SEP)

################################ OPTIONS ####################################

set(BOARD "S6E2CC8" CACHE STRING "The Board to use (S6E2CC8 [default], S6E2CC9 or S6E2CCA)")
set(AVAILABLE_BOARDS "S6E2CC8" "S6E2CC9" "S6E2CCA")
set(CACHE BOARD PROPERTY STRINGS ${AVAILABLE_BOARDS})
if(NOT BOARD IN_LIST AVAILABLE_BOARDS)
  message(FATAL_ERROR "Invalid board selected: '${BOARD}', select one from '${AVAILABLE_BOARDS}'")
endif()

set(CONFIGURATION "release" CACHE STRING "Configuration to use (release [default] or debug)")
if(NOT CONFIGURATION STREQUAL "release" OR CONFIGURATION STREQUAL "debug")
  message(FATAL_ERROR "Invalid Configuration selected: '${CONFIGURATION}', select either 'release' or 'debug'")
endif()

################################ TARGETS ####################################

string(TOLOWER ${BOARD} LBOARD)

# direct copy from winidea, no clue why those specifically are different
if(${BOARD} STREQUAL "S6E2CCA")
  set(BOARD_SPECIFIC_LIBM "m")
endif()

set(TARGET "${BOARD}_${CONFIGURATION}")

if(${CONFIGURATION} STREQUAL "release")
  set(LINKER_SCRIPT "linker/${LBOARD}_rom.ld")
else()
  set(LINKER_SCRIPT "linker/${LBOARD}_ram.ld")
endif()
cmake_path(ABSOLUTE_PATH LINKER_SCRIPT)

add_library(${TARGET}_objs OBJECT ${SOURCES}) #needs to be done to get the right linking order
add_executable(${TARGET})
target_link_libraries(${TARGET} PUBLIC ${TARGET}_objs ${BOARD_SPECIFIC_LIBM} "c" "nosys")

set(LINKER_OPTIONS ${COMMON_LINKER_OPTIONS}
  "--script=${LINKER_SCRIPT}"
  "--output=$<TARGET_FILE:${TARGET}>"
  "-Map=$<TARGET_FILE_DIR:${TARGET}>/$<TARGET_FILE_BASE_NAME:${TARGET}>.map"
)
list(JOIN LINKER_OPTIONS "," _LINKER_OPTIONS_JOINED)
target_link_options(${TARGET} PRIVATE
  ${COMMON_COMPILER_LINK_OPTIONS} 
  "LINKER:${_LINKER_OPTIONS_JOINED}"
)

add_custom_command(TARGET ${TARGET} POST_BUILD 
  COMMAND ${ARM_NONE_EABI_OBJCOPY}
  "-O" "srec"
  "$<TARGET_FILE:${TARGET}>"
  "$<TARGET_FILE_DIR:${TARGET}>/$<TARGET_FILE_BASE_NAME:${TARGET}>.srec"
)
