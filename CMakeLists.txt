cmake_minimum_required(VERSION 3.19.0)
project(mpv-cross C)
include(ProcessorCount)
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/download_externalproject.cmake)
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/custom_steps.cmake)

cmake_policy(SET CMP0097 NEW)
cmake_policy(SET CMP0114 NEW)
if(POLICY CMP0135)
  cmake_policy(SET CMP0135 NEW)
endif()

if(NOT CMAKE_GENERATOR STREQUAL "Ninja")
    message(WARNING "Generator \"${CMAKE_GENERATOR}\" is unsupported!\nTry Ninja if you encounter problems.")
endif()

# Supported arch: x86_64-w64-mingw32 , i686-w64-mingw32 , aarch64-w64-mingw32
set(TARGET_ARCH "x86_64-w64-mingw32" CACHE STRING "The target architecture for this toolchain.")
string(REGEX MATCH "^[^-]+" TARGET_CPU ${TARGET_ARCH})

execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpmachine OUTPUT_VARIABLE HOST_ARCH OUTPUT_STRIP_TRAILING_WHITESPACE)
string(REGEX MATCH "^[^-]+" HOST_CPU ${HOST_ARCH})

execute_process(COMMAND meson --version OUTPUT_VARIABLE MESON_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
if(NOT MESON_VERSION GREATER_EQUAL 1.3)
    message(FATAL_ERROR "Meson 1.3.0 or greater is required. Install it with pip instead.")
endif()

if(TARGET_CPU STREQUAL "x86_64")
    set(TARGET_CPU_FAMILY "x86_64")
    set(TARGET_BITS 64)
    set(GCC_ARCH "x86-64" CACHE STRING "Target architecture for x86_64")
    string(REGEX MATCH "-v3" x86_64_LEVEL ${GCC_ARCH})
elseif(TARGET_CPU STREQUAL "i686")
    set(TARGET_CPU_FAMILY "x86")
    set(TARGET_BITS 32)
    set(GCC_ARCH "prescott" CACHE STRING "Target architecture for x86")
elseif(TARGET_CPU STREQUAL "aarch64")
    set(TARGET_CPU_FAMILY "aarch64")
    set(GCC_ARCH "cortex-a76" CACHE STRING "Target cpu for aarch64")
endif()

processorcount(CPU_COUNT)
if(CPU_COUNT EQUAL 0)
    set(CPU_COUNT 1)
endif()

# Improved malloc library detection with permission checks
if(EXISTS /usr/lib/libmimalloc.so AND IS_READABLE /usr/lib/libmimalloc.so)
    set(MALLOC "/usr/lib/libmimalloc.so" CACHE STRING "set malloc implementation")
elseif(EXISTS /usr/lib/libjemalloc.so AND IS_READABLE /usr/lib/libjemalloc.so)
    set(MALLOC "/usr/lib/libjemalloc.so" CACHE STRING "set malloc implementation")
endif()

if(DEFINED MALLOC AND EXISTS ${MALLOC} AND IS_READABLE ${MALLOC})
    option(LLD_LTO_ALL_THREADS "Allow LLD to use all threads for ThinLTO, non-Glibc malloc implementations can improve performance." ON)
else()
    unset(MALLOC CACHE)
endif()

set(MAKEJOBS ${CPU_COUNT})
# set(MAKEJOBS 5) # uncomment this line to manually change no. of cpu
set(MAKEJOBS ${MAKEJOBS} CACHE STRING "Number of parallel processes to run for each package build")

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    # Create install directory with proper permissions if it doesn't exist
    set(_default_install_prefix "${CMAKE_CURRENT_BINARY_DIR}/install")
    if(NOT EXISTS ${_default_install_prefix})
        file(MAKE_DIRECTORY ${_default_install_prefix})
    endif()
    set(CMAKE_INSTALL_PREFIX ${_default_install_prefix} CACHE PATH "Default install path" FORCE)
endif()

set(MINGW_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX}/${TARGET_ARCH} CACHE STRING "Target toolchain path")
set(SINGLE_SOURCE_LOCATION "" CACHE STRING "Repository or tarball stored path")
set(RUSTUP_LOCATION "${CMAKE_CURRENT_BINARY_DIR}/install_rustup" CACHE STRING "Rust toolchain path")

# Ensure rustup directory exists and is writable
if(NOT EXISTS ${RUSTUP_LOCATION})
    file(MAKE_DIRECTORY ${RUSTUP_LOCATION})
endif()

set(LLVM_ENABLE_LTO "OFF" CACHE STRING "OFF, ON, Thin and Full")
set(LLVM_ENABLE_PGO "OFF" CACHE STRING "OFF, GEN, CSGEN, USE")

if(LLVM_ENABLE_PGO STREQUAL "GEN" OR LLVM_ENABLE_PGO STREQUAL "CSGEN")
    set(LLVM_PROFILE_DATA_DIR "${CMAKE_INSTALL_PREFIX}/profiles/" CACHE PATH "Default profile generation directory")
    # Ensure profile directory exists and is writable
    if(NOT EXISTS ${LLVM_PROFILE_DATA_DIR})
        file(MAKE_DIRECTORY ${LLVM_PROFILE_DATA_DIR})
    endif()
endif()

if(LLVM_ENABLE_PGO STREQUAL "USE" OR LLVM_ENABLE_PGO STREQUAL "CSGEN")
    set(LLVM_PROFDATA_FILE "" CACHE FILEPATH "Profdata file path")
    if(DEFINED LLVM_PROFDATA_FILE AND NOT (EXISTS ${LLVM_PROFDATA_FILE} AND IS_READABLE ${LLVM_PROFDATA_FILE}))
        set(LLVM_ENABLE_PGO OFF)
    endif()
endif()

option(CLANG_PACKAGES_LTO "Build each package with ThinLTO." OFF)
option(ENABLE_CCACHE "Enable ccache" OFF)

if(ENABLE_CCACHE)
    set(CCACHE_MAXSIZE "500M" CACHE STRING "Size of ccache")
    set(CCACHE_DIR "${MINGW_INSTALL_PREFIX}/ccache" CACHE STRING "Directory to keep ccached data")
    
    # Ensure ccache directory exists and is writable
    if(NOT EXISTS ${CCACHE_DIR})
        file(MAKE_DIRECTORY ${CCACHE_DIR})
    endif()
    
    find_program(CCACHE_EXISTS ccache NO_CACHE)
    if(CCACHE_EXISTS)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/ccache_conf.in ${CCACHE_DIR}/ccache.conf @ONLY)
    else()
        set(ENABLE_CCACHE OFF)
    endif()
endif()

option(LLVM_CCACHE_BUILD "Enable ccache" OFF)
if(LLVM_CCACHE_BUILD)
    set(LLVM_CCACHE_MAXSIZE "500M" CACHE STRING "Size of ccache")
    set(LLVM_CCACHE_DIR "${CMAKE_INSTALL_PREFIX}/llvm-ccache" CACHE STRING "Directory to keep ccached data")
    # Ensure llvm-ccache directory exists and is writable
    if(NOT EXISTS ${LLVM_CCACHE_DIR})
        file(MAKE_DIRECTORY ${LLVM_CCACHE_DIR})
    endif()
endif()

# Configure files with proper permission checks
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/toolchain.cmake.in)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/toolchain.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/toolchain.cmake @ONLY)
endif()
set(TOOLCHAIN_FILE ${CMAKE_CURRENT_BINARY_DIR}/toolchain.cmake)

set(CLANG_FLAGS "" CACHE STRING "These flags will be added to the end of the clang args")
set(LLD_FLAGS "" CACHE STRING "These flags will be added to the end of the lld args")
set(GCC_FLAGS "" CACHE STRING "These flags will be added to the end of the gcc args")
set(BFD_FLAGS "" CACHE STRING "These flags will be added to the end of the bfd args")
set(COMPILER_TOOLCHAIN "gcc" CACHE STRING "gcc or clang")

if(COMPILER_TOOLCHAIN STREQUAL "gcc")
    if(TARGET_ARCH STREQUAL "aarch64-w64-mingw32")
        message(FATAL_ERROR "The target architecture is unavailable for this toolchain.")
    endif()
endif()

# Configure meson cross file with permission check
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/meson_cross.txt.in)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/meson_cross.txt.in ${CMAKE_CURRENT_BINARY_DIR}/meson_cross.txt @ONLY)
endif()
set(MESON_CROSS ${CMAKE_CURRENT_BINARY_DIR}/meson_cross.txt)

# Configure cargo config with permission checks
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/cargo_config.in)
    # Ensure .cargo directory exists
    set(CARGO_CONFIG_DIR "${RUSTUP_LOCATION}/.cargo")
    if(NOT EXISTS ${CARGO_CONFIG_DIR})
        file(MAKE_DIRECTORY ${CARGO_CONFIG_DIR})
    endif()
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cargo_config.in ${CARGO_CONFIG_DIR}/config @ONLY)
endif()

# Configure exec file with permission check
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/exec.in)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/exec.in ${CMAKE_CURRENT_BINARY_DIR}/exec @ONLY)
    # Make exec file executable
    file(CHMOD ${CMAKE_CURRENT_BINARY_DIR}/exec PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE)
endif()
set(EXEC ${CMAKE_CURRENT_BINARY_DIR}/exec)
set(MAKE ${EXEC} make -j${MAKEJOBS})

if(TARGET_CPU STREQUAL "x86_64")
    execute_process(
        COMMAND ${EXEC} echo | ${CMAKE_C_COMPILER} -dM -E - -march=${GCC_ARCH}
        OUTPUT_VARIABLE CC_OUTPUT
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    if(CC_OUTPUT MATCHES "__AVX")
        set(GCC_ARCH_HAS_AVX ON)
    endif()
    if(CC_OUTPUT MATCHES "__AVX512F")
        set(GCC_ARCH_HAS_AVX512 ON)
    endif()
endif()

string(TIMESTAMP BUILDDATE "%Y%m%d" UTC)

# Configure publish file with permission check
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/publish.in)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/publish.in ${CMAKE_CURRENT_BINARY_DIR}/publish @ONLY)
    # Make publish file executable if needed
    file(CHMOD ${CMAKE_CURRENT_BINARY_DIR}/publish PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE)
endif()

message(STATUS "MINGW_INSTALL_PREFIX dir: ${MINGW_INSTALL_PREFIX}")
message(STATUS "SINGLE_SOURCE_LOCATION dir: ${SINGLE_SOURCE_LOCATION}")
message(STATUS "RUSTUP_LOCATION dir: ${RUSTUP_LOCATION}")
message(STATUS "Compiler used: ${COMPILER_TOOLCHAIN}")
message(STATUS "Malloc: ${MALLOC}")
message(STATUS "Ccache: ${ENABLE_CCACHE}")
message(STATUS "Target toolchain to build: ${TARGET_ARCH}")

add_subdirectory(toolchain)
add_subdirectory(packages)
