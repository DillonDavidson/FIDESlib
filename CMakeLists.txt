cmake_minimum_required(VERSION 3.25.2)
cmake_policy(VERSION 3.21.3...3.28)

project(FIDESlib VERSION 0.1 DESCRIPTION "Homomorphic Encryption Library")

############### Project configuration ###################
set(FIDESLIB_CONFIGURATION_TYPES "Release;Debug;RelWithDebInfo" CACHE STRING "FIDESlib configuration types.")
if (NOT DEFINED FIDESLIB_BUILD_TYPE)
    set(FIDESLIB_BUILD_TYPE "Release")
endif ()

############### Build Type Validation ###################

if (NOT FIDESLIB_BUILD_TYPE IN_LIST FIDESLIB_CONFIGURATION_TYPES)
    message(FATAL_ERROR "Invalid build type: ${FIDESLIB_BUILD_TYPE}")
endif ()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

############### Utils ###################

include(FetchContent)
include(CMakePrintHelpers)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

############### CPP ###################

enable_language(CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

############### CUDA ##################

include(CheckLanguage)
check_language(CUDA)

# Defaulting to 89 since my system uses it
set(CMAKE_CUDA_ARCHITECTURES 89)

if (NOT DEFINED CUDA_TOOLKIT_ROOT_DIR)
    set(CUDA_TOOLKIT_ROOT_DIR /usr/local/cuda)
endif ()

set(CMAKE_CUDA_COMPILER ${CUDA_TOOLKIT_ROOT_DIR}/bin/nvcc)

enable_language(CUDA)

set(CMAKE_RELOCATABLE_DEVICE_CODE ON)
set(CMAKE_CUDA_SEPARABLE_COMPILATION ON)
set(CMAKE_CUDA_STANDARD 20)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_EXTENSIONS ON)

find_package(CUDAToolkit REQUIRED)

############### OpenFHE Configuration ###################

# Default runs will not update nor install OpenFHE. Specify manually to install.
find_package(OpenFHE CONFIG HINTS ${FIDESLIB_INSTALL_OPENFHE_PREFIX} REQUIRED)

option(BUILD_STATIC "Set to ON to include static versions of the library" OFF)

if (OpenFHE_FOUND)
    message(STATUS "FOUND PACKAGE OpenFHE")
    message(STATUS "OpenFHE Version: ${BASE_OPENFHE_VERSION}")
    message(STATUS "OpenFHE installed as shared libraries: ${OpenFHE_SHARED}")
    message(STATUS "OpenFHE include files location: ${OpenFHE_INCLUDE}")
    message(STATUS "OpenFHE lib files location: ${OpenFHE_LIBDIR}")
    message(STATUS "OpenFHE Native Backend size: ${OpenFHE_NATIVE_SIZE}")
else ()
    message(FATAL_ERROR "PACKAGE OpenFHE NOT FOUND")
endif ()

set(CMAKE_CXX_FLAGS ${OpenFHE_CXX_FLAGS})
set(CMAKE_CUDA_FLAGS "-I${OpenFHE_INCLUDE}/..")

cmake_print_variables(OpenFHE_CXX_FLAGS)
cmake_print_variables(CMAKE_CUDA_FLAGS)

include_directories(${OPENMP_INCLUDES})
include_directories(${OpenFHE_INCLUDE})
include_directories(${OpenFHE_INCLUDE}/third-party/include)
include_directories(${OpenFHE_INCLUDE}/core)
include_directories(${OpenFHE_INCLUDE}/pke)
include_directories(${OpenFHE_INCLUDE}/binfhe)

link_directories(${OpenFHE_LIBDIR})
link_directories(${OPENMP_LIBRARIES})

############### NVTX ###################

include(nvtx/c/nvtxImportedTargets.cmake)

############### FIDESlib Definition ###################

set(SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)
file(GLOB_RECURSE SOURCE_FILES ${SOURCE_DIR}/*.cpp ${SOURCE_DIR}/*.cu)
file(GLOB_RECURSE HEADER_FILES ${INCLUDE_DIR}/*.h ${INCLUDE_DIR}/*.cuh ${INCLUDE_DIR}/*.hpp)

add_library(fideslib STATIC)

target_sources(fideslib PRIVATE ${SOURCE_FILES})
target_sources(fideslib PUBLIC FILE_SET HEADERS BASE_DIRS ${INCLUDE_DIR} FILES ${HEADER_FILES})

target_include_directories(fideslib
        PRIVATE nvtx/c/include
)

target_link_libraries(fideslib
        PUBLIC CUDA::cudart
        PUBLIC tbb
        PUBLIC ${OpenFHE_LIBDIR}/libOPENFHEcore_static.a
        PUBLIC ${OpenFHE_LIBDIR}/libOPENFHEpke_static.a
        PUBLIC ${OpenFHE_LIBDIR}/libOPENFHEbinfhe_static.a
        PUBLIC ${OpenFHE_LIBDIR}/libOPENFHEhook_static.a
)

target_compile_options(fideslib PRIVATE
        $<$<COMPILE_LANGUAGE:CXX>:-Wall -ltbb -O3>
        $<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CONFIG:DEBUG>>:-g -O0 -rdynamic>
        $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=-Wall,-lineinfo,-ltbb,-O3 -rdc=true -ltbb>
        $<$<AND:$<COMPILE_LANGUAGE:CUDA>,$<CONFIG:DEBUG>>:-G>
        $<$<AND:$<COMPILE_LANGUAGE:CUDA>,$<CONFIG:RELWITHDEBINFO>>:>
)

set_target_properties(fideslib PROPERTIES
        CUDA_SEPARABLE_COMPILATION ON
        POSITION_INDEPENDENT_CODE ON
        CUDA_RESOLVE_DEVICE_SYMBOLS ON
        PREFIX ""
)
