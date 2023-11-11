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

add_subdirectory(oldmin/ext/argon2)

add_library(argon2-gpu-common SHARED
oldmin/lib/argon2-gpu-common/argon2params.cpp
oldmin/lib/argon2-gpu-common/blake2b.cpp
)
target_include_directories(argon2-gpu-common INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/oldmin/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-gpu-common PRIVATE
${CMAKE_CURRENT_SOURCE_DIR}/oldmin/include/argon2-gpu-common
${CMAKE_CURRENT_SOURCE_DIR}/oldmin/lib/argon2-gpu-common
)

if(CUDA_FOUND)
    cuda_add_library(argon2-cuda SHARED
    oldmin/lib/argon2-cuda/cudaexception.cpp
    oldmin/lib/argon2-cuda/device.cpp
    oldmin/lib/argon2-cuda/globalcontext.cpp
    oldmin/lib/argon2-cuda/programcontext.cpp
    oldmin/lib/argon2-cuda/processingunit.cpp
    oldmin/lib/argon2-cuda/kernelrunner.cu
    )
else()
    add_library(argon2-cuda SHARED
    oldmin/lib/argon2-cuda/nocuda.cpp
    )
endif()

target_include_directories(argon2-cuda PRIVATE
oldmin/include/argon2-cuda
oldmin/lib/argon2-cuda
)
target_include_directories(argon2-cuda INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/oldmin/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(argon2-cuda argon2-gpu-common)

add_library(argon2-opencl SHARED
oldmin/lib/argon2-opencl/device.cpp
oldmin/lib/argon2-opencl/globalcontext.cpp
oldmin/lib/argon2-opencl/kernelloader.cpp
oldmin/lib/argon2-opencl/programcontext.cpp
oldmin/lib/argon2-opencl/processingunit.cpp
oldmin/lib/argon2-opencl/kernelrunner.cpp
)
target_include_directories(argon2-opencl INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/oldmin/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-opencl PRIVATE
oldmin/include/argon2-opencl
oldmin/lib/argon2-opencl
)
target_link_libraries(argon2-opencl
    argon2-gpu-common -lOpenCL
)

add_executable(oldmingup
oldmin/src/argon2-gpu-bench/cpuexecutive.cpp
oldmin/src/argon2-gpu-bench/cudaexecutive.cpp
oldmin/src/argon2-gpu-bench/benchmark.cpp
oldmin/src/argon2-gpu-bench/main.cpp
)
target_include_directories(oldmingup PRIVATE oldmin/src/argon2-gpu-bench)
target_link_libraries(oldmingup
    argon2-cuda argon2
)

install(
    TARGETS argon2-gpu-common argon2-cuda
    DESTINATION ${LIBRARY_INSTALL_DIR}
)
install(FILES
oldmin/include/argon2-gpu-common/argon2-common.h
oldmin/include/argon2-gpu-common/argon2params.h
oldmin/include/argon2-opencl/cl.hpp
oldmin/include/argon2-opencl/opencl.h
oldmin/include/argon2-opencl/device.h
oldmin/include/argon2-opencl/globalcontext.h
oldmin/include/argon2-opencl/programcontext.h
oldmin/include/argon2-opencl/processingunit.h
oldmin/include/argon2-opencl/kernelrunner.h
oldmin/include/argon2-cuda/cudaexception.h
oldmin/include/argon2-cuda/kernelrunner.h
oldmin/include/argon2-cuda/device.h
oldmin/include/argon2-cuda/globalcontext.h
oldmin/include/argon2-cuda/programcontext.h
oldmin/include/argon2-cuda/processingunit.h
    DESTINATION ${INCLUDE_INSTALL_DIR}
)
install(
    TARGETS oldmingup
    DESTINATION ${BINARY_INSTALL_DIR}
)
