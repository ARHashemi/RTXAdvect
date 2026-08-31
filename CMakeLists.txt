# ======================================================================== #
# Copyright 2019-2020 The Collaborators                                    #
# Ported to CUDA 13.x + OptiX 9.x + NVIDIA/OWL by JAXTrace revision team,  #
# 2026-08-28.                                                              #
# Licensed under the Apache License, Version 2.0 (see LICENSE).            #
# ======================================================================== #

cmake_minimum_required(VERSION 3.24)
project(RTXAdvect VERSION 0.0.2 LANGUAGES C CXX CUDA)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ------------------------------------------------------------------
# NVIDIA/OWL as a subproject.  New OWL exports targets `owl::owl`
# and provides the `embed_ptx` CMake function via its cmake/ dir.
# ------------------------------------------------------------------
set(owl_dir ${PROJECT_SOURCE_DIR}/owl)
add_subdirectory(${owl_dir} EXCLUDE_FROM_ALL)

# embed_ptx() lives in owl/owl/cmake/embed_ptx.cmake; add_subdirectory
# above should already have brought it into scope, but include the
# module path explicitly for clarity.
list(APPEND CMAKE_MODULE_PATH "${owl_dir}/owl/cmake")
include(embed_ptx)

# ------------------------------------------------------------------
# OptixTetQueries library
# ------------------------------------------------------------------
embed_ptx(
  OUTPUT_TARGET
    optixQueryKernel_ptx
  PTX_LINK_LIBRARIES
    owl::owl
  EMBEDDED_SYMBOL_NAMES
    ptxCode
  SOURCES
    optix/optixQueryKernel.cu
)

add_library(optixTetQueries STATIC
  optix/internalTypes.h
  optix/OptixQuery.h
  optix/OptixTetQuery.cpp
  optix/OptixTriQuery.cpp
)
target_include_directories(optixTetQueries PUBLIC ${PROJECT_SOURCE_DIR})
target_link_libraries(optixTetQueries
  PUBLIC
    owl::owl
    optixQueryKernel_ptx
)

# ------------------------------------------------------------------
# The main executable
# ------------------------------------------------------------------
add_executable(cudaParticleAdvection
  cuda/common.h
  cuda/utils.cpp
  cuda/particles.cu
  cuda/HostTetMesh.h
  cuda/DeviceTetMesh.cuh
  query/ConvexQuery.h
  query/ConvexQuery.cu
  query/RTQuery.h
  query/RTQuery.cu
  cuda/cudaParticleAdvection.cu
)
target_include_directories(cudaParticleAdvection PRIVATE ${PROJECT_SOURCE_DIR})
target_link_libraries(cudaParticleAdvection
  PRIVATE
    optixTetQueries
    owl::owl
)

# RTX 5090 = Blackwell / sm_120 is picked up by CMAKE_CUDA_ARCHITECTURES
# passed on the command line; OWL sets sensible defaults if unset.

message(STATUS "RTXAdvect ported to NVIDIA/OWL + OptiX 9.x + CUDA 13.x")

# double4-deprecated shim — CUDA 13 renamed double4 to double4_16a /
# double4_32a but kept the original as a deprecated alias.  Silence the
# warning here rather than patching every function signature that
# transitively takes double4.
target_compile_options(optixTetQueries      PRIVATE -Wno-deprecated-declarations)
target_compile_options(cudaParticleAdvection PRIVATE -Wno-deprecated-declarations)

# nvcc 13 promotes some warnings; relax constexpr host->device calls
# (std::min/std::max on the device side) and silence printf format
# checks (there are a few benign %d vs long int mismatches in the
# original code).
target_compile_options(optixTetQueries       PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:--expt-relaxed-constexpr>
    $<$<COMPILE_LANGUAGE:CXX>:-Wno-format>)
target_compile_options(cudaParticleAdvection PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:--expt-relaxed-constexpr>
    $<$<COMPILE_LANGUAGE:CXX>:-Wno-format>)
