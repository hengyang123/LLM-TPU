cmake_minimum_required(VERSION 3.10)
project(llama2)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON CACHE INTERNAL "") 

if (NOT DEFINED TARGET_ARCH)
    set(TARGET_ARCH pcie)
endif()

include_directories(${PROJECT_SOURCE_DIR}/../support/include)

if (${CMAKE_HOST_SYSTEM_PROCESSOR} STREQUAL "aarch64")
	add_definitions(-DSOC_TARGET)
	link_directories(${PROJECT_SOURCE_DIR}/../support/lib_soc)
	message("SoC mode, starting......")
elseif (${TARGET_ARCH} STREQUAL "pcie")
    add_definitions(-DPCIE_TARGET)
    link_directories(${PROJECT_SOURCE_DIR}/../support/lib_pcie)
	message("Pcie mode, starting......")
elseif (${TARGET_ARCH} STREQUAL "soc")
    add_definitions(-DSOC_TARGET)
    set(CMAKE_C_COMPILER /opt/aarch64-linux-gnu-7.5.0/bin/aarch64-linux-gnu-gcc)
    set(CMAKE_ASM_COMPILER /opt/aarch64-linux-gnu-7.5.0/bin/aarch64-linux-gnu-gcc)
    set(CMAKE_CXX_COMPILER /opt/aarch64-linux-gnu-7.5.0/bin/aarch64-linux-gnu-g++)
    link_directories(${PROJECT_SOURCE_DIR}/../support/lib_soc)
	message("SoC mode, starting......")
endif()

add_definitions(-DDEBUG --std=c++17 -fPIC -Wall -Werror)
set(CMAKE_BUILD_TYPE "Debug")

add_executable(llama2 demo.cpp)
target_link_libraries(llama2 bmlib bmrt sentencepiece)
