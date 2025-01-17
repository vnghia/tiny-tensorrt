cmake_minimum_required(VERSION 3.22)

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  add_compile_options(-fdiagnostics-color=always)
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  add_compile_options(-fcolor-diagnostics)
endif()

# Project
project(
  tinytrt
  VERSION 1.1.0
  LANGUAGES CXX
)
enable_language(CUDA)
set(PROJECT_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR})
list(APPEND CMAKE_MODULE_PATH "${PROJECT_ROOT_DIR}/cmake")

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()

# Prevent building in the source directory
if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(
    FATAL_ERROR
    "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there."
  )
endif()

# Ccache
option(ENABLE_CCACHE "Automatically use ccache if available" ON)

if("${ENABLE_CCACHE}")
  find_program(CCACHE_PROGRAM ccache NAMES /usr/bin/ccache)

  if(CCACHE_PROGRAM)
    message(STATUS "ccache found: ${CCACHE_PROGRAM}")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    set(CMAKE_CC_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
  endif()
endif()

# Output directory
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_ROOT_DIR}/lib/${CMAKE_BUILD_TYPE}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_ROOT_DIR}/lib/${CMAKE_BUILD_TYPE}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_ROOT_DIR}/bin/${CMAKE_BUILD_TYPE}")

# Build option
option(TRT_BUILD_PYTHON "Compile tiny-tensorrt python api" OFF)
option(TRT_BUILD_SAMPLE "Build tiny-tensorrt samples" OFF)

# Zlib
find_package(ZLIB REQUIRED)

# spdlog
include(external/spdlog)

# CUDA
find_package(CUDAToolkit REQUIRED)

# TensorRT
find_library(LIBNVINFER NAME nvinfer HINTS ${TENSORRT_LIB_PATH} REQUIRED)
find_library(LIBNVINFER_PLUGIN NAME nvinfer_plugin HINTS ${TENSORRT_LIB_PATH} REQUIRED)
find_library(LIBNVPARSERS NAME nvparsers HINTS ${TENSORRT_LIB_PATH} REQUIRED)
find_path(TENSORRT_INCLUDE_DIR NvInfer.h HINTS ${TENSORRT_INCLUDE_PATH})

# Final config
message(STATUS "
=============> Final Config <===============
TensorRT headers: ${TENSORRT_INCLUDE_DIR}
TensorRT library: ${LIBNVINFER}
============================================
")

# Tiny TensorRT
add_library(${PROJECT_NAME}
  src/cnpy.cpp
  src/Int8Calibrator.cu
  src/Trt.cu
)
target_compile_features(${PROJECT_NAME} PUBLIC cuda_std_11)

target_compile_options(${PROJECT_NAME} PUBLIC -Wall -Wno-deprecated -Wfloat-conversion)
set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${PROJECT_VERSION})

target_include_directories(${PROJECT_NAME} PUBLIC
  ${PROJECT_SOURCE_DIR}/
  ${PROJECT_SOURCE_DIR}/src
  ${spdlog_SOURCE_DIR}/include
  ${TENSORRT_INCLUDE_DIR}
)
target_link_libraries(${PROJECT_NAME} PUBLIC
  ZLIB::ZLIB
  CUDA::cudart
  nvinfer
  nvinfer_plugin
  nvparsers
)

# Sample
if(TRT_BUILD_SAMPLE)
  add_subdirectory(samples)
endif()
