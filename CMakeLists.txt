# Copyright (c) 2019-2023 Advanced Micro Devices, Inc. All rights reserved.
# Modifications Copyright (c) Microsoft Corporation. Licensed under the MIT License.

# CMake version minimum requirements
#==================================================================================================
cmake_minimum_required(VERSION 3.5)

# CMake Toolchain file to define compilers and path to ROCm
#==================================================================================================
if (NOT CMAKE_TOOLCHAIN_FILE)
  set(CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_SOURCE_DIR}/toolchain-linux.cmake")
  message(STATUS "CMAKE_TOOLCHAIN_FILE: ${CMAKE_TOOLCHAIN_FILE}")
endif()

# RCCL project
#==================================================================================================
project(rccl CXX)

# Build options
#==================================================================================================
option(BUILD_ADDRESS_SANITIZER                 "Enable address sanitizer"                      OFF)
option(BUILD_BFD                               "Enable custom backtrace (if bfd.h exists)"     OFF)
option(BUILD_FILE_REORG_BACKWARD_COMPATIBILITY "File/folder reorg with backward compatibility" OFF)
option(BUILD_LOCAL_GPU_TARGET_ONLY             "Build only for GPUs detected on this machine"  OFF)
option(BUILD_SHARED_LIBS                       "Build as shared library"                       ON)
option(BUILD_TESTS                             "Build unit test programs"                      OFF)
option(COLLTRACE                               "Collective Trace Option"                       ON)
option(ENABLE_MSCCL_KERNEL                     "Enable MSCCL while compiling"                  ON)
option(ENABLE_IFC                              "Enable indirect function call"                 OFF)
option(INSTALL_DEPENDENCIES                    "Force install dependencies"                    OFF)
option(ROCTX                                   "Enable ROCTX"                                  OFF)
option(PROFILE                                 "Enable profiling"                              OFF)
option(TIMETRACE                               "Enable time-trace during compilation"          OFF)
option(TRACE                                   "Enable additional tracing"                     OFF)

# Default GPU architectures to build
#==================================================================================================
set(DEFAULT_GPUS
      gfx803
      gfx900:xnack-
      gfx906:xnack-
      gfx908:xnack-
      gfx90a:xnack-
      gfx90a:xnack+
      gfx940
      gfx941
      gfx942
      gfx1030
      gfx1100
      gfx1101
      gfx1102)

# Load CMake modules
#==================================================================================================
include(CheckIncludeFiles)
include(CheckSymbolExists)
include(cmake/Dependencies.cmake) # GTest, rocm-cmake, rocm_local_targets
include(cmake/Generator.cmake)    # Configure functions that goes into RCCL

# Build only for local GPU architecture
if (BUILD_LOCAL_GPU_TARGET_ONLY)
  message(STATUS "Building only for local GPU target")
  if (COMMAND rocm_local_targets)
    rocm_local_targets(DEFAULT_GPUS)
  else()
    message(WARNING "Unable to determine local GPU targets. Falling back to default GPUs.")
  endif()
endif()

# Determine which GPU architectures to build for
set(AMDGPU_TARGETS "${DEFAULT_GPUS}" CACHE STRING "Target default GPUs if AMDGPU_TARGETS is not defined.")
if (COMMAND rocm_check_target_ids)
  message(STATUS "Checking for ROCm support for GPU targets:")
  rocm_check_target_ids(SUPPORTED_GPUS TARGETS ${AMDGPU_TARGETS})
else()
  message(WARNING "Unable to check for supported GPU targets. Falling back to default GPUs.")
  set(SUPPORTED_GPUS ${DEFAULT_GPUS})
endif()
set(GPU_TARGETS "${SUPPORTED_GPUS}" CACHE STRING "GPU targets to compile for.")
message(STATUS "Compiling for ${GPU_TARGETS}")

## NOTE: Reload rocm-cmake in order to update GPU_TARGETS
include(cmake/Dependencies.cmake) # Reloading to use desired GPU_TARGETS instead of defaults

# Try to establish ROCM_PATH (for find_package)
#==================================================================================================
if(NOT DEFINED ROCM_PATH)
  if("${CMAKE_CXX_COMPILER}" MATCHES ".*hipcc$")
    # Guess based on provided compiler location
    get_filename_component(_real_path ${CMAKE_CXX_COMPILER} REALPATH)
    get_filename_component(_new_path "${_real_path}" DIRECTORY)
    get_filename_component(ROCM_PATH "${_new_path}/.." REALPATH)
    message(STATUS "Setting ROCM_PATH based on hipcc location to ${ROCM_PATH}")
  else()
    # Guess default location
    set(ROCM_PATH "/opt/rocm")
    message(WARNING "Unable to find ROCM_PATH: Falling back to ${ROCM_PATH}")
  endif()
else()
  message(STATUS "ROCM_PATH found: ${ROCM_PATH}")
endif()

# Set CMAKE flags
#==================================================================================================
set(CMAKE_INSTALL_PREFIX "${ROCM_PATH}" CACHE PATH "")
set(CMAKE_CXX_STANDARD   14)   # We use C++14 features, this will add compile option: -std=c++14
set(CMAKE_CXX_EXTENSIONS OFF)  # Without this line, it will add -std=gnu++14 instead, which has some issues.
list(APPEND CMAKE_PREFIX_PATH  # Add ROCM_PATH to CMake search paths (for finding HIP / HSA
            ${ROCM_PATH}
            ${ROCM_PATH}/hip
            ${ROCM_PATH}/llvm
            ${ROCM_PATH}/hcc)

# Check for required dependencies
#==================================================================================================
## Check for Threads
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

## Check for HIP
find_package(hip REQUIRED)
message(STATUS "HIP compiler:     ${HIP_COMPILER}")
message(STATUS "HIP runtime:      ${HIP_RUNTIME}")
if(NOT "${HIP_COMPILER}" MATCHES "clang")
  message(FATAL_ERROR "RCCL requires clang-based compiler (hipcc)")
endif()
find_program(hipcc_executable hipcc)
message(STATUS "hipcc executable: ${hipcc_executable}")
execute_process(
  COMMAND         bash "-c" "${hipcc_executable} --version | grep 'HIP version' | awk -F\" \" '{ printf $3}' | awk -F\"-\" '{ printf $1}'"
  OUTPUT_VARIABLE hipcc_version_string)
message(STATUS "hipcc version:    ${hipcc_version_string}")

## Check for ROCm version
execute_process(
  COMMAND         bash "-c" "cat ${ROCM_PATH}/.info/version"
  OUTPUT_VARIABLE rocm_version_string
)
string(REGEX MATCH "([0-9]+)\\.([0-9]+)\\.([0-9]+)" rocm_version_matches ${rocm_version_string})
if (rocm_version_matches)
    set(ROCM_MAJOR_VERSION ${CMAKE_MATCH_1})
    set(ROCM_MINOR_VERSION ${CMAKE_MATCH_2})
    set(ROCM_PATCH_VERSION ${CMAKE_MATCH_3})

    message(STATUS "ROCm version: ${ROCM_MAJOR_VERSION}.${ROCM_MINOR_VERSION}.${ROCM_PATCH_VERSION}")

    # Convert the version components to int for comparison
    math(EXPR ROCM_VERSION "(10000 * ${ROCM_MAJOR_VERSION}) + (100 * ${ROCM_MINOR_VERSION}) + ${ROCM_PATCH_VERSION}")
    add_definitions("-DROCM_VERSION=${ROCM_VERSION}")
else()
    message(WARNING "Failed to extract ROCm version.")
endif()

### Check for hipDeviceMallocUncached support
check_symbol_exists("hipDeviceMallocUncached" "hip/hip_runtime_api.h" HIP_UNCACHED_MEMORY)

### Check for hipDeviceMallocContiguous support
check_symbol_exists("hipDeviceMallocContiguous" "hip/hip_runtime_api.h" HIP_CONTIGUOUS_MEMORY)

### Check for indirect function call support
if(ENABLE_IFC)
  if(${hipcc_version_string} VERSION_GREATER_EQUAL "5.5.30201")
    set(IFC_ENABLED ON)
    message(STATUS "Indirect function call enabled")
  else()
    set(IFC_ENABLED OFF)
    message(WARNING "Indirect function call disabled - requires hipcc version >= 5.5.30201")
  endif()
endif()

## Check for hsa-runtime64
find_package(hsa-runtime64 REQUIRED)
get_target_property(HSA_INCLUDE_PATH hsa-runtime64::hsa-runtime64 INTERFACE_INCLUDE_DIRECTORIES)
message(STATUS "HSA runtime: ${HSA_INCLUDE_PATH}")

## Check for ROCM-smi
find_package(rocm_smi PATHS ${ROCM_PATH}/lib/cmake/rocm_smi)
if (rocm_smi_FOUND)
  message(STATUS "Found rocm_smi at ${ROCM_SMI_INCLUDE_DIR}")
else()
  message(STATUS "Checking old include directory structure for rocm_smi")
  set(ROCM_SMI_INCLUDE_DIR "${ROCM_PATH}/rocm_smi/include")
  set(ROCM_SMI_LIB_DIR     "${ROCM_PATH}/rocm_smi/lib")
  set(ROCM_SMI_LIBRARIES   rocm_smi64)
endif()
check_include_file_cxx("${ROCM_SMI_INCLUDE_DIR}/rocm_smi/rocm_smi64Config.h" HAVE_ROCM_SMI64CONFIG)
### Check for RSMI_INIT_FLAG_THRAD_ONLY_MUTEX support
file(READ "${ROCM_SMI_INCLUDE_DIR}/rocm_smi/rocm_smi.h" rocm_smi_incl)
string(FIND "${rocm_smi_incl}" "RSMI_INIT_FLAG_THRAD_ONLY_MUTEX" matchres)
if(${matchres} EQUAL -1)
  message(STATUS "RSMI_INIT_FLAG_THRAD_ONLY_MUTEX not supported")
else()
  message(STATUS "RSMI_INIT_FLAG_THRAD_ONLY_MUTEX supported")
  set(HAVE_ROCM_SMI_THREAD_ONLY_MUTEX True)
endif ()

## Check for BFD library if custom backtrace is requested
if(BUILD_BFD)
  enable_language(C)
  check_include_files(bfd.h HAVE_BFD)
  if (HAVE_BFD)
    message(STATUS "-- Found BFD support")

    # Check for specific BFD feature support
    CHECK_SYMBOL_EXISTS(bfd_get_section_flags "bfd.h" HAVE_DECL_BFD_GET_SECTION_FLAGS)
    CHECK_SYMBOL_EXISTS(bfd_get_section_vma   "bfd.h" HAVE_DECL_BFD_GET_SECTION_VMA)
    CHECK_CXX_SOURCE_COMPILES(
      "#include <bfd.h>

       int main (int argc, char **argv){
           bfd_size_type size;
           bfd abfd;
           asection sec;
           size = bfd_section_size(&abfd, &sec);
           return (int)(size);
       }"
      HAVE_TWO_ARG_BFD_SECTION_SIZE)

    # Check for iberty support
    find_library(HAVE_IBERTY iberty PATHS /usr/lib64 /usr/lib/ PATH_SUFFIXES x86_64-linux-gnu)
    if(HAVE_IBERTY)
      message(STATUS "iberty found @ ${HAVE_IBERTY}")
    endif()

    # Check for demangle support
    find_path(DEMANGLE_DIR demangle.h PATHS /usr/include PATH_SUFFIXES libiberty)
    if(NOT DEMANGLE_DIR)
      message(WARNING "Could not find demangle.h ${DEMANGLE_DIR}")
    else()
      message(STATUS "Found demangle.h in ${DEMANGLE_DIR}")
    endif()
  else()
    message(WARNING "bfd.h header not found - Disabling custom backtrace")
  endif()
endif()

# Check for --amdgpu-kernarg-preload-count
check_cxx_compiler_flag("-mllvm --amdgpu-kernarg-preload-count=16" HAVE_KERNARG_PRELOAD)
if (HAVE_KERNARG_PRELOAD)
  message(STATUS "Kernarg preloading to SGPR enabled")
endif()

# Determine version from makefiles/version.mk and fill in templates
#==================================================================================================
## parse version from Makefile NCCL_MAJOR, NCCL_MINOR, NCCL_PATCH must exist
## NCCL_SUFFIX is optional
## NCCL_VERSION formatting is ((X) * 1000 + (Y) * 100 + (Z)) so we must first detect one or two digits first
file(READ makefiles/version.mk version_mk_text)
if("${version_mk_text}" MATCHES "NCCL_MAJOR *:= *([0-9]*)")
  set(NCCL_MAJOR ${CMAKE_MATCH_1})
else()
  message(FATAL_ERROR "Failed to parse NCCL_MAJOR")
endif()
if("${version_mk_text}" MATCHES "NCCL_MINOR *:= *([0-9]*)")
  set(NCCL_MINOR ${CMAKE_MATCH_1})
else()
  message(FATAL_ERROR "Failed to parse NCCL_MINOR")
endif()
if("${version_mk_text}" MATCHES "NCCL_PATCH *:= *([0-9]*)")
  set(NCCL_PATCH ${CMAKE_MATCH_1})
else()
  message(FATAL_ERROR "Failed to parse NCCL_PATCH")
endif()
if("${version_mk_text}" MATCHES "NCCL_SUFFIX *:= *([0-9]*)")
  set(NCCL_SUFFIX ${CMAKE_MATCH_1})
else()
  set(NCCL_SUFFIX)
endif()
if("${version_mk_text}" MATCHES "PKG_REVISION *:= *([0-9]*)")
  set(PKG_REVISION ${CMAKE_MATCH_1})
else()
  message(FATAL_ERROR "Failed to parse PKG_REVISION")
endif()
if("${NCCL_PATCH}" MATCHES "[0-9][0-9]")
  set(NCCL_VERSION "${NCCL_MAJOR}${NCCL_MINOR}${NCCL_PATCH}")
else()
  set(NCCL_VERSION "${NCCL_MAJOR}${NCCL_MINOR}0${NCCL_PATCH}")
endif()

## Setup VERSION
set(VERSION_STRING "${NCCL_MAJOR}.${NCCL_MINOR}.${NCCL_PATCH}")
rocm_setup_version(VERSION ${VERSION_STRING})

## Fill in version information for main header file
configure_file(src/nccl.h.in ${PROJECT_BINARY_DIR}/include/rccl/rccl.h) # For external linking
configure_file(src/nccl.h.in ${PROJECT_BINARY_DIR}/include/nccl.h)      # Used by some internal files

# Collect list of all source files
#==================================================================================================
# E.g: find src -type f \( -name "*.cc" -o -name "*.h" -o -name "*.hpp" \) | sort
set(SRC_FILES
  src/bootstrap.cc
  src/channel.cc
  src/collectives.cc
  src/debug.cc
  src/enqueue.cc
  src/group.cc
  src/init.cc
  src/init_nvtx.cc
  src/net.cc
  src/msccl.cc
  src/proxy.cc
  src/register.cc
  src/transport.cc
# src/clique/AllReduceCliqueKernel.h
# src/clique/CliqueCommon.h
# src/clique/CliqueManager.cc
# src/clique/CliqueManager.h
# src/clique/CliqueShmNames.h
# src/clique/HandleCache.cc
# src/clique/HandleCache.h
# src/clique/HandleShm.cc
# src/clique/HandleShm.h
# src/clique/Hash.cc
# src/clique/Hash.h
# src/clique/MsgQueue.cc
# src/clique/MsgQueue.h
# src/clique/SharedMemHelper.h
# src/clique/ShmObject.cc
# src/clique/ShmObject.h
  src/device/all_gather.h
  src/device/all_reduce.h
  src/device/alltoall_pivot.h
  src/device/broadcast.h
  src/device/common.h
  src/device/common_kernel.h
  src/device/op128.h
  src/device/primitives.h
  src/device/prims_ll128.h
  src/device/prims_ll.h
  src/device/prims_simple.h
  src/device/reduce.h
  src/device/reduce_kernel.h
  src/device/reduce_scatter.h
  src/device/sendrecv.h
  src/device/common.cu
  src/device/onerank.cu
  src/device/network/unpack/unpack_defs.h
  src/device/network/unpack/unpack.h
  src/graph/connect.cc
  src/graph/paths.cc
  src/graph/rings.cc
  src/graph/rings.h
  src/graph/rome_models.cc
  src/graph/rome_models.h
  src/graph/search.cc
  src/graph/topo.cc
  src/graph/topo.h
  src/graph/trees.cc
  src/graph/tuning.cc
  src/graph/xml.cc
  src/graph/xml.h
  src/include/align.h
  src/include/alloc.h
  src/include/archinfo.h
  src/include/argcheck.h
  src/include/BfdBacktrace.hpp
  src/include/bootstrap.h
  src/include/channel.h
  src/include/checks.h
  src/include/collectives.h
  src/include/coll_net.h
  src/include/comm.h
  src/include/core.h
  src/include/cpuset.h
# src/include/cudawrap.h
  src/include/debug.h
  src/include/device.h
  src/include/enqueue.h
  src/include/gdrwrap.h
  src/include/git_version.h
  src/include/graph.h
  src/include/group.h
  src/include/ibvcore.h
  src/include/ibvsymbols.h
  src/include/ibvwrap.h
  src/include/info.h
  src/include/ipcsocket.h
  src/include/nccl_common.h
  src/include/nccl_net.h
  src/include/nccl_tuner.h
  src/include/net_device.h
  src/include/net.h
  src/include/nvmlwrap.h
  src/include/nvtx.h
  src/include/nvtx_stub.h
  src/include/p2p.h
  src/include/param.h
  src/include/profiler.h
  src/include/proxy.h
  src/include/rccl_vars.h
  src/include/register.h
  src/include/rccl_float8.h
  src/include/rocm_smi_wrap.h
  src/include/rocmwrap.h
  src/include/roctx.h
  src/include/shm.h
  src/include/signals.h
  src/include/socket.h
  src/include/strongstream.h
  src/include/timer.h
  src/include/transport.h
  src/include/trees.h
  src/include/tuner.h
  src/include/utils.h
  src/include/msccl/msccl_lifecycle.h
  src/include/msccl/msccl_parser.h
  src/include/msccl/msccl_scheduler.h
  src/include/msccl/msccl_setup.h
  src/include/msccl/msccl_status.h
  src/include/msccl/msccl_struct.h
  src/include/npkit/npkit.h
  src/include/npkit/npkit_event.h
  src/include/npkit/npkit_struct.h
  src/include/nvtx3/nvToolsExtCuda.h
  src/include/nvtx3/nvToolsExtCudaRt.h
  src/include/nvtx3/nvToolsExt.h
  src/include/nvtx3/nvToolsExtOpenCL.h
  src/include/nvtx3/nvToolsExtPayload.h
  src/include/nvtx3/nvToolsExtSync.h
  src/include/nvtx3/nvtx3.hpp
  src/include/nvtx3/nvtxDetail/nvtxImplCore.h
  src/include/nvtx3/nvtxDetail/nvtxImplCudaRt_v3.h
  src/include/nvtx3/nvtxDetail/nvtxImplCuda_v3.h
  src/include/nvtx3/nvtxDetail/nvtxImpl.h
  src/include/nvtx3/nvtxDetail/nvtxImplOpenCL_v3.h
  src/include/nvtx3/nvtxDetail/nvtxImplSync_v3.h
  src/include/nvtx3/nvtxDetail/nvtxInitDecls.h
  src/include/nvtx3/nvtxDetail/nvtxInitDefs.h
  src/include/nvtx3/nvtxDetail/nvtxInit.h
  src/include/nvtx3/nvtxDetail/nvtxLinkOnce.h
  src/include/nvtx3/nvtxDetail/nvtxTypes.h
  src/include/nvtx3/nvtxExtDetail/nvtxExtImpl.h
  src/include/nvtx3/nvtxExtDetail/nvtxExtImplPayload_v1.h
  src/include/nvtx3/nvtxExtDetail/nvtxExtInit.h
  src/include/nvtx3/nvtxExtDetail/nvtxExtPayloadTypeInfo.h
  src/include/nvtx3/nvtxExtDetail/nvtxExtTypes.h
  src/include/alt_rsmi.h
  src/misc/alt_rsmi.cc
  src/misc/archinfo.cc
  src/misc/argcheck.cc
# src/misc/cudawrap.cc
# src/misc/gdrwrap.cc
  src/misc/ibvsymbols.cc
  src/misc/ibvwrap.cc
  src/misc/ipcsocket.cc
  src/misc/npkit.cc
# src/misc/nvmlwrap.cc
  src/misc/nvmlwrap_stub.cc
  src/misc/param.cc
  src/misc/profiler.cc
  src/misc/rocm_smi_wrap.cc
  src/misc/rocmwrap.cc
  src/misc/roctx.cc
  src/misc/shmutils.cc
  src/misc/signals.cc
  src/misc/socket.cc
  src/misc/strongstream.cc
  src/misc/tuner.cc
  src/misc/utils.cc
  src/misc/msccl/msccl_lifecycle.cc
  src/misc/msccl/msccl_parser.cc
  src/misc/msccl/msccl_setup.cc
  src/misc/msccl/msccl_status.cc
  src/transport/coll_net.cc
  src/transport/net.cc
  src/transport/net_ib.cc
  src/transport/net_socket.cc
  src/transport/nvls.cc
  src/transport/p2p.cc
  src/transport/shm.cc
)

if (ENABLE_MSCCL_KERNEL)
  set(MSCCL_KERNEL_SOURCES
    src/device/msccl_kernel_impl.h
    src/include/msccl/msccl_kernel.h
  )
  list(APPEND SRC_FILES ${MSCCL_KERNEL_SOURCES})
endif()

# Hipify source files (copy of source generated into hipify directory)
#==================================================================================================
find_program(hipify-perl_executable hipify-perl)
set(HIPIFY_DIR "${CMAKE_CURRENT_BINARY_DIR}/hipify")

## Loop over each source file to hipify
foreach(SRC_FILE ${SRC_FILES})
  # Check that file exists
  if (NOT EXISTS ${CMAKE_SOURCE_DIR}/${SRC_FILE})
    message(FATAL_ERROR "Unable to find file listed in CMakeLists.txt: ${CMAKE_SOURCE_DIR}/${SRC_FILE}")
  endif()

  # Establish hipified copy of the source file
  set(HIP_FILE "${HIPIFY_DIR}/${SRC_FILE}")
  get_filename_component(HIP_FILE_DIR ${HIP_FILE} DIRECTORY)

  # Make sure the file name is unique and there is no duplicate
  add_file_unique(HIP_SOURCES ${HIP_FILE})

  # Convert .cu files to .cpp so that they get processed properly
  string(REPLACE "\.cu" "\.cu.cpp" HIP_FILE ${HIP_FILE})
  list(APPEND HIP_SOURCES ${HIP_FILE})

  # Create a custom command to create hipified source code
  add_custom_command(
    OUTPUT ${HIP_FILE}
    COMMAND mkdir -p ${HIP_FILE_DIR} && $ ${hipify-perl_executable} -quiet-warnings ${CMAKE_SOURCE_DIR}/${SRC_FILE} -o ${HIP_FILE}
    MAIN_DEPENDENCY ${SRC_FILE}
    COMMENT "Hipifying ${SRC_FILE} -> ${HIP_FILE}"
  )
endforeach()

# Generate device/host tables and all the collective functions that are going to be in librccl.so
#==================================================================================================
if(ONLY_FUNCS)
  ## Generate only the specified functions
  gen_functions(${ONLY_FUNCS})
else()
  # Generate all the functions
  gen_functions("AllGather|AllReduce|AllToAllPivot|Broadcast|Reduce|ReduceScatter|SendRecv")
endif()

# Create an initial git_version.cpp file (that will be updated with latest git version)
#==================================================================================================
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/git_version.cpp "")
list(APPEND HIP_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/git_version.cpp)

# Create a custom target that updates git_version.cpp and executes whenever rccl is built
add_custom_target(git_version_check
  COMMENT "Updating git_version.cpp if necessary"
  COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/git_version.cmake
  VERBATIM
)

# Set up RCCL library
#==================================================================================================
## Set RCCL source files
add_library(rccl ${HIP_SOURCES})

## Set RCCL dependencies
add_dependencies(rccl git_version_check)                                      # Execute git_version_check during build

## Set RCCL include directories
target_include_directories(rccl PRIVATE ${PROJECT_BINARY_DIR}/include)        # for generated rccl.h header
target_include_directories(rccl PRIVATE ${HIPIFY_DIR}/src)                    # for hipfied headers
target_include_directories(rccl PRIVATE ${HIPIFY_DIR}/src/include)
target_include_directories(rccl PRIVATE ${HIPIFY_DIR}/src/device)
target_include_directories(rccl PRIVATE ${HIPIFY_DIR}/src/device/network/unpack)
target_include_directories(rccl PRIVATE ${HSA_INCLUDE_PATH})
target_include_directories(rccl PRIVATE ${ROCM_SMI_INCLUDE_DIR})
if(DEMANGLE_DIR)
  target_include_directories(rccl PRIVATE ${DEMANGLE_DIR})
endif()

## Set RCCL compile definitions
if(COLLTRACE)
  target_compile_definitions(rccl PRIVATE ENABLE_COLLTRACE)
endif()
if(ENABLE_MSCCL_KERNEL)
  target_compile_definitions(rccl PRIVATE COMPILE_MSCCL_KERNEL)
endif()
if(HAVE_ROCM_SMI64CONFIG)
  target_compile_definitions(rccl PRIVATE USE_ROCM_SMI64CONFIG)
endif()
if(HAVE_ROCM_SMI_THREAD_ONLY_MUTEX)
  target_compile_definitions(rccl PRIVATE USE_ROCM_SMI_THREAD_ONLY_MUTEX)
endif()
if(NPKIT_FLAGS)
  target_compile_definitions(rccl PRIVATE ${NPKIT_FLAGS})
endif()
if(PROFILE)
  target_compile_definitions(rccl PRIVATE ENABLE_PROFILING)
endif()
if(NOT ROCTX)
  target_compile_definitions(rccl PRIVATE NVTX_NO_IMPL)
  target_compile_definitions(rccl PRIVATE ROCTX_NO_IMPL)
  target_compile_definitions(rccl PRIVATE NVTX_DISABLE)
endif()
if(TRACE)
  target_compile_definitions(rccl PRIVATE ENABLE_TRACE)
endif()
if(${HIP_CONTIGUOUS_MEMORY})
  target_compile_definitions(rccl PRIVATE HIP_CONTIGUOUS_MEMORY)
endif()
if(${hipcc_version_string} VERSION_GREATER_EQUAL "5.7.31920")
  target_compile_definitions(rccl PRIVATE HIP_UNCACHED_MEMORY)
  message(STATUS "HIP_UNCACHED_MEMORY enabled")
else()
  # keep --hipcc-func-supp on older HIP and compiler
  if(NOT IFC_ENABLED)
    target_compile_options(rccl PRIVATE --hipcc-func-supp)
  endif()
endif()
if (BUILD_BFD)
  if (HAVE_BFD)
    target_compile_definitions(rccl PRIVATE HAVE_BFD)
  endif()
  if (HAVE_DECL_BFD_GET_SECTION_FLAGS)
    target_compile_definitions(rccl PRIVATE HAVE_DECL_BFD_GET_SECTION_FLAGS)
  endif()
  if (HAVE_DECL_BFD_GET_SECTION_VMA)
    target_compile_definitions(rccl PRIVATE HAVE_DECL_BFD_GET_SECTION_VMA)
  endif()
  if (HAVE_TWO_ARG_BFD_SECTION_SIZE)
    target_compile_definitions(rccl PRIVATE HAVE_TWO_ARG_BFD_SECTION_SIZE)
  endif()
endif()
if (IFC_ENABLED)
  target_compile_definitions(rccl PRIVATE USE_INDIRECT_FUNCTION_CALL)
endif()
if(DEMANGLE_DIR)
  target_compile_definitions(rccl PRIVATE "HAVE_CPLUS_DEMANGLE=1")
  target_compile_definitions(rccl PRIVATE "HAVE_DECL_BASENAME=1")
endif()
if(${hipcc_version_string} VERSION_GREATER_EQUAL "6.1.33591")
  set(LL128_ENABLED ON)
	target_compile_definitions(rccl PRIVATE ENABLE_LL128)
	message(STATUS "RCCL LL128 protocol enabled")
endif()

## Set RCCL compile options
target_compile_options(rccl PRIVATE -parallel-jobs=12)
target_compile_options(rccl PRIVATE -Werror=uninitialized -Werror=sometimes-uninitialized)
target_compile_options(rccl PRIVATE -Wno-format-nonliteral)
target_compile_options(rccl PRIVATE -fgpu-rdc)               # Generate relocatable device code (required for extern __shared__)
target_compile_options(rccl PRIVATE -fvisibility=hidden)     # Set symbol visibility to hidden by default
if (HAVE_KERNARG_PRELOAD)
  target_compile_options(rccl PRIVATE -mllvm --amdgpu-kernarg-preload-count=16)
endif()

## NOTE: This is currently being handled by rocm-cmake, however may need to be re-enabled in the future
#foreach(target ${GPU_TARGETS})
#  target_compile_options(rccl PRIVATE --offload-arch=${target})
#endforeach()

if(BUILD_ADDRESS_SANITIZER)
  target_compile_options(rccl PRIVATE -fsanitize=address -shared-libasan)
endif()
if(TIMETRACE)
  target_compile_options(rccl PRIVATE -ftime-trace)
endif()

## Set RCCL linked library directories
target_link_directories(rccl PRIVATE ${ROCM_SMI_LIB_DIR})

## Set RCCL linked libraries
if (HAVE_BFD)
  target_link_libraries(rccl PRIVATE bfd)
  if(HAVE_IBERTY)
    target_link_libraries(rccl PRIVATE iberty z)
  endif()
endif()
if (ROCTX)
  target_link_libraries(rccl PRIVATE -lroctx64)
endif()
target_link_libraries(rccl PRIVATE -fgpu-rdc)             # Required when linking relocatable device code
target_link_libraries(rccl PRIVATE   Threads::Threads)
target_link_libraries(rccl INTERFACE hip::host)
target_link_libraries(rccl PRIVATE   hip::device)
target_link_libraries(rccl PRIVATE   dl)
target_link_libraries(rccl PRIVATE   ${ROCM_SMI_LIBRARIES})

## Set RCCL link options
target_link_options(rccl PRIVATE -parallel-jobs=16)       # Use multiple threads to link
if(BUILD_ADDRESS_SANITIZER)
  target_link_options(rccl PRIVATE -fuse-ld=lld)
endif()
if(TIMETRACE)
  target_link_options(rccl PRIVATE -ftime-trace)
endif()

if(NOT BUILD_SHARED_LIBS)
  message(STATUS "Building static RCCL library")
else()
  message(STATUS "Building shared RCCL library")
endif()
if (HAVE_KERNARG_PRELOAD)
  target_link_options(rccl PRIVATE -Xoffload-linker -mllvm=-amdgpu-kernarg-preload-count=16)
endif()

## Track linking time
set_property(TARGET rccl PROPERTY RULE_LAUNCH_LINK "${CMAKE_COMMAND} -E time")

## Setup librccl.so version
rocm_set_soversion(rccl "1.0")

# Install settings
#==================================================================================================
## Specify install targets
rocm_install_targets(TARGETS rccl)
rocm_install(FILES       ${PROJECT_BINARY_DIR}/include/rccl/rccl.h src/include/nccl_net.h
             DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/rccl)
file(COPY tools/msccl-algorithms DESTINATION ${PROJECT_BINARY_DIR})
file(COPY tools/msccl-unit-test-algorithms DESTINATION ${PROJECT_BINARY_DIR})
## Install Algorithm files under share folder
rocm_install(DIRECTORY ${PROJECT_BINARY_DIR}/msccl-algorithms DESTINATION ${CMAKE_INSTALL_DATADIR}/rccl)
rocm_install(DIRECTORY ${PROJECT_BINARY_DIR}/msccl-unit-test-algorithms DESTINATION ${CMAKE_INSTALL_DATADIR}/rccl)

rocm_export_targets(
  NAMESPACE roc::
  TARGETS   rccl
  DEPENDS   hip)

## Build with backwards compatibility if requested
if(BUILD_FILE_REORG_BACKWARD_COMPATIBILITY)
  ### Create wrapper files
  rocm_wrap_header_dir(
    "${PROJECT_BINARY_DIR}/include/rccl"
    PATTERNS "rccl.h"
    GUARDS SYMLINK WRAPPER
    WRAPPER_LOCATIONS ${CMAKE_INSTALL_INCLUDEDIR} rccl/${CMAKE_INSTALL_INCLUDEDIR})

  ### install the wrapper header file to package
  rocm_install(
    FILES ${PROJECT_BINARY_DIR}/rccl/include/rccl.h src/include/nccl_net.h
    DESTINATION "./rccl/${CMAKE_INSTALL_INCLUDEDIR}/" )
  rocm_install(
    FILES ${PROJECT_BINARY_DIR}/include/rccl.h src/include/nccl_net.h
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/" )
endif()

## Set package dependencies
if(BUILD_ADDRESS_SANITIZER)
  set(DEPENDS_HIP_RUNTIME "hip-runtime-amd-asan" )
else()
  set(DEPENDS_HIP_RUNTIME "hip-runtime-amd" )
endif()
rocm_package_add_dependencies(DEPENDS "${DEPENDS_HIP_RUNTIME} >= 4.5.0" "rocm-smi-lib >= 4.0.0")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "/opt" "${ROCM_PATH}")

find_file (DEBIAN debian_version debconf.conf PATHS /etc)
if(DEBIAN)
  # Write copyright file
  file(WRITE "${CMAKE_BINARY_DIR}/copyright"
  "Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Upstream-Name: rccl
Source: https://github.com/ROCm/rccl

Files: *
Copyright: (c) 2016-2020, NVIDIA CORPORATION. All rights reserved.
Modifications Copyright (c) 2020-2023 Advanced Micro Devices, Inc. All rights reserved.
Modifications Copyright (c) Microsoft Corporation. Licensed under the MIT License.
License: See LICENSE.txt for license information\n")
  rocm_install(FILES "${CMAKE_BINARY_DIR}/copyright" DESTINATION ${CMAKE_INSTALL_DATADIR}/rccl)
  # Write changelog file
  find_program( date_executable date )
  execute_process(COMMAND ${date_executable} -R OUTPUT_VARIABLE TIMESTAMP)
  file(WRITE "${CMAKE_BINARY_DIR}/changelog"
  "rccl (${VERSION_STRING}-1) unstable; urgency=medium

  * Initial release.

 -- RCCL Maintainer <rccl-maintainer@amd.com>  ${TIMESTAMP}\n")
  find_program( gzip_executable gzip )
  execute_process(COMMAND bash "-c" "${gzip_executable} -9 -c ${CMAKE_BINARY_DIR}/changelog"
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR} OUTPUT_FILE "${CMAKE_BINARY_DIR}/changelog.Debian.gz")
  rocm_install(FILES "${CMAKE_BINARY_DIR}/changelog.Debian.gz" DESTINATION ${CMAKE_INSTALL_DATADIR}/rccl)
  set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "ROCm Communication Collectives Library
  Optimized primitives for collective multi-GPU communication")
endif()

if(BUILD_TESTS)
  rocm_package_setup_component(clients)
  rocm_package_setup_client_component(tests PACKAGE_NAME unittests)
  add_subdirectory(test)

  if(BUILD_SHARED_LIBS)
    add_custom_command(TARGET rccl POST_BUILD
      COMMENT "Extracting metadata from librccl.so"
      COMMAND COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/extract_metadata.cmake
      VERBATIM
    )
  endif()
endif()

rocm_create_package(
  NAME        rccl
  DESCRIPTION "ROCm Communication Collectives Library"
  MAINTAINER  "RCCL Maintainer <rccl-maintainer@amd.com>"
  LDCONFIG)
