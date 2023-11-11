cmake_minimum_required(VERSION 3.7)

project(argon2-gpu CXX)
set(BINARY_INSTALL_DIR /usr/local/bin)
set(LIBRARY_INSTALL_DIR /usr/local/lib)
set(INCLUDE_INSTALL_DIR /usr/local/include)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR})

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CUDA_ARCH "sm_75" CACHE STRING "Specify the CUDA architecture.")

set(CUDA_FOUND FALSE)

if(NOT NO_CUDA)
    find_package(CUDA)
endif()

if(CUDA_FOUND)
    message("INFO: Using CUDA version ${CUDA_VERSION}")
    add_definitions(-DHAVE_CUDA=1)
else()
    message("INFO: Building without CUDA support")
    add_definitions(-DHAVE_CUDA=0)
endif()

if(CUDA_FOUND)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};
        -std=c++11;-O3;--ptxas-options=-v;-arch ${CUDA_ARCH};-lineinfo
    )
endif()

add_subdirectory(oldminz/ext/argon2)

add_library(argon2-gpu-common SHARED
oldminz/lib/argon2-gpu-common/argon2params.cpp
oldminz/lib/argon2-gpu-common/blake2b.cpp
)
target_include_directories(argon2-gpu-common INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/oldminz/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-gpu-common PRIVATE
${CMAKE_CURRENT_SOURCE_DIR}/oldminz/include/argon2-gpu-common
${CMAKE_CURRENT_SOURCE_DIR}/oldminz/lib/argon2-gpu-common
)

if(CUDA_FOUND)
    cuda_add_library(argon2-cuda SHARED
    oldminz/lib/argon2-cuda/cudaexception.cpp
    oldminz/lib/argon2-cuda/device.cpp
    oldminz/lib/argon2-cuda/globalcontext.cpp
    oldminz/lib/argon2-cuda/programcontext.cpp
    oldminz/lib/argon2-cuda/processingunit.cpp
    oldminz/lib/argon2-cuda/kernelrunner.cu
    )
else()
    add_library(argon2-cuda SHARED
    oldminz/lib/argon2-cuda/nocuda.cpp
    )
endif()

target_include_directories(argon2-cuda PRIVATE
oldminz/include/argon2-cuda
oldminz/lib/argon2-cuda
)
target_include_directories(argon2-cuda INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/oldminz/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(argon2-cuda argon2-gpu-common)

add_library(argon2-opencl SHARED
oldminz/lib/argon2-opencl/device.cpp
oldminz/lib/argon2-opencl/globalcontext.cpp
oldminz/lib/argon2-opencl/kernelloader.cpp
oldminz/lib/argon2-opencl/programcontext.cpp
oldminz/lib/argon2-opencl/processingunit.cpp
oldminz/lib/argon2-opencl/kernelrunner.cpp
)
target_include_directories(argon2-opencl INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/oldminz/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-opencl PRIVATE
oldminz/include/argon2-opencl
oldminz/lib/argon2-opencl
)
target_link_libraries(argon2-opencl
    argon2-gpu-common -lOpenCL
)

add_executable(oldminzgup
oldminz/src/argon2-gpu-bench/cpuexecutive.cpp
oldminz/src/argon2-gpu-bench/cudaexecutive.cpp
oldminz/src/argon2-gpu-bench/benchmark.cpp
oldminz/src/argon2-gpu-bench/main.cpp
)
target_include_directories(oldminzgup PRIVATE oldminz/src/argon2-gpu-bench)
target_link_libraries(oldminzgup
    argon2-cuda argon2
)

install(
    TARGETS argon2-gpu-common argon2-cuda
    DESTINATION ${LIBRARY_INSTALL_DIR}
)
install(FILES
oldminz/include/argon2-gpu-common/argon2-common.h
oldminz/include/argon2-gpu-common/argon2params.h
oldminz/include/argon2-opencl/cl.hpp
oldminz/include/argon2-opencl/opencl.h
oldminz/include/argon2-opencl/device.h
oldminz/include/argon2-opencl/globalcontext.h
oldminz/include/argon2-opencl/programcontext.h
oldminz/include/argon2-opencl/processingunit.h
oldminz/include/argon2-opencl/kernelrunner.h
oldminz/include/argon2-cuda/cudaexception.h
oldminz/include/argon2-cuda/kernelrunner.h
oldminz/include/argon2-cuda/device.h
oldminz/include/argon2-cuda/globalcontext.h
oldminz/include/argon2-cuda/programcontext.h
oldminz/include/argon2-cuda/processingunit.h
    DESTINATION ${INCLUDE_INSTALL_DIR}
)
install(
    TARGETS oldminzgup
    DESTINATION ${BINARY_INSTALL_DIR}
)
