cmake_minimum_required(VERSION 3.0)
project(ISM CXX)

set(CMAKE_CXX_STANDARD 20)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release"
            CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
            FORCE)
endif (NOT CMAKE_BUILD_TYPE)

# set(WARNINGS "-Wall -Wextra -Wshadow -Wdouble-promotion -Wuninitialized -Wsign-promo -Wredundant-decls -Winit-self -Wmissing-include-dirs -Wdisabled-optimization -Wctor-dtor-privacy")#-Wsign-conversion
# set(DISABLED_WARNINGS "-Wno-assume -Wno-unused-command-line-argument -Wno-tautological-constant-compare -Wno-unknown-warning-option -Wno-linker-warnings -Wno-nonportable-include-path")
# set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Ofast -fomit-frame-pointer -march=native -mtune=native -s ${DISABLED_WARNINGS} -DNDEBUG")
# set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=undefined ${WARNINGS} ${DISABLED_WARNINGS}")
# set(CMAKE_SHARED_LINKER_FLAGS "-dead_strip")

#if (APPLE)
#    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Xpreprocessor -fopenmp -I/usr/local/include -L/usr/local/lib -lomp")
#else ()
#    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
#endif ()

find_package(OpenMP REQUIRED)

#Flags for DPC++. For hipSYCL use -DHIPSYCL_TARGETS=omp;cuda:sm_xx;...
if (SYCL STREQUAL CUDA)
    set(DPCPP_FLAGS -fsycl-targets=nvptx64-nvidia-cuda,spir64 -Xsycl-target-backend=nvptx64-nvidia-cuda --cuda-gpu-arch=sm_75 -Wno-unknown-cuda-version -Xcuda-ptxas -v -fgpu-inline-threshold=10000)
elseif (SYCL STREQUAL HIP)
    set(DPCPP_FLAGS -fsycl-targets=amdgcn-amd-amdhsa -Xsycl-target-backend --offload-arch=gfx902 -fsycl-id-queries-fit-in-int -fgpu-inline-threshold=1000)
elseif (SYCL STREQUAL CPU)
    set(DPCPP_FLAGS -fsycl-targets=spir64_x86_64)
endif ()

#In all the cases we need SYCl for the vector class used in the OMP backend...
include(cmake/FindSYCL.cmake)

option(BUILD_DOUBLE "Build with double backend" ON)
option(BUILD_FLOAT "Build with float backend" ON)
option(BUILD_HALF "Build with half backend" OFF)

if (BUILD_DOUBLE)
    add_definitions(-DBUILD_DOUBLE)
endif ()
if (BUILD_FLOAT)
    add_definitions(-DBUILD_FLOAT)
endif ()
if (BUILD_HALF)
    add_definitions(-DBUILD_HALF)
endif ()


include_directories(include)

add_executable(main main.cpp)
add_sycl_to_target(TARGET main SOURCES main.cpp)

add_executable(main_exam main_exam.cpp)
add_sycl_to_target(TARGET main_exam SOURCES main_exam.cpp)

# OMP BE
add_library(md_cpu STATIC src/molecular_dynamics_cpu.cpp)
add_sycl_to_target(TARGET md_cpu SOURCES src/molecular_dynamics_cpu.cpp)
target_compile_options(md_cpu PRIVATE "-fvisibility=hidden")

if (OpenMP_CXX_FOUND)
    target_link_libraries(md_cpu PRIVATE OpenMP::OpenMP_CXX)
else ()
    message(STATUS "OpenMP not found!")
endif ()

target_link_libraries(main PRIVATE md_cpu)
target_link_libraries(main_exam PRIVATE md_cpu)

install(TARGETS main main_exam)

# SYCL BE
if (SYCL)
    add_definitions(-DBUILD_SYCL)
    add_library(md_sycl STATIC src/molecular_dynamics_sycl.cpp)
    add_sycl_to_target(TARGET md_sycl SOURCES src/molecular_dynamics_sycl.cpp)
    target_compile_options(md_sycl PRIVATE "-fvisibility=hidden")
    target_link_libraries(main PRIVATE md_sycl)
    target_link_libraries(main_exam PRIVATE md_sycl)
endif ()

include(FetchContent)

if (BUILD_BENCH)
    # Externally provided libraries
    FetchContent_Declare(googletest
            GIT_REPOSITORY https://github.com/google/googletest.git
            GIT_TAG main)

    FetchContent_Declare(googlebenchmark
            GIT_REPOSITORY https://github.com/google/benchmark.git
            GIT_TAG main) # need master for benchmark::benchmark

    FetchContent_MakeAvailable(
            googletest
            googlebenchmark)

    include(benchmarks/CMakeLists.txt)
endif ()
