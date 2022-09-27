CMAKE_MINIMUM_REQUIRED(VERSION 3.1)
PROJECT(PerformanceLab)

SET_PROPERTY(GLOBAL PROPERTY USE_FOLDERS ON)

SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeModules" ${CMAKE_MODULE_PATH})

SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# Enable C++11 for host code
set(CMAKE_CXX_STANDARD 11)

# Set a default build type if none was specified
IF(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    SET(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui
    SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
ENDIF()

########################################
# CUDA Setup
########################################
FIND_PACKAGE(CUDA 10 REQUIRED)
INCLUDE(${CMAKE_MODULE_PATH}/CUDAComputesList.cmake)

FIND_LIBRARY(CUDA_nvToolsExt_LIBRARY
    NAMES "nvToolsExt" "nvToolsExt64_1"
    PATH_SUFFIXES "lib64" "lib" "lib/x64" "lib/Win32"
    PATHS ${CUDA_TOOLKIT_ROOT_DIR}
          $ENV{NVTOOLSEXT_PATH}
    DOC "CUDA NV Tools Ext Library"
  )

IF(WIN32)
    FIND_PATH(
        CUDA_nvToolsExt_INCLUDE
        NAMES "nvToolsExt.h"
        PATHS $ENV{NVTOOLSEXT_PATH}
        PATH_SUFFIXES "include"
        DOC "NV Tools Ext Include Directory"
        NO_DEFAULT_PATH
    )

    INCLUDE_DIRECTORIES(${CUDA_nvToolsExt_INCLUDE})

    # Set up include and lib paths
    set(CUDA_HOST_COMPILER ${CMAKE_CXX_COMPILER} CACHE FILEPATH "Host side compiler used by NVCC" FORCE)
ENDIF(WIN32)

LIST(APPEND CUDA_NVCC_FLAGS ${CUDA_GENERATE_CODE})
LIST(APPEND CUDA_NVCC_FLAGS_DEBUG "-g -G")
SET(CUDA_VERBOSE_BUILD ON)

########################################

INCLUDE_DIRECTORIES(
    ${CMAKE_INCLUDE_PATH}
    ${CMAKE_CURRENT_SOURCE_DIR}/source
    ${CUDA_INCLUDE_DIRS}
    )

########################################
# Setup the executables
########################################
SET(EXECUTABLES transpose reduction reductionrecursive)
FILE(GLOB headers source/*.h source/*.hpp)
FOREACH(exe ${EXECUTABLES})
    CUDA_ADD_EXECUTABLE(${exe} source/${exe}.cu ${headers})
    TARGET_LINK_LIBRARIES(${exe} ${CUDA_nvToolsExt_LIBRARY})

    # Copy nvToolsExt DLL as it is not in PATH
    IF(WIN32)
        ADD_CUSTOM_COMMAND(TARGET ${exe}
            POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different "\"$ENV{NVTOOLSEXT_PATH}bin\\x64\\nvToolsExt64_1.dll\"" $<TARGET_FILE_DIR:${exe}>
            COMMENT "Copying nvToolsExt DLL"
        )
    ENDIF(WIN32)
ENDFOREACH()
########################################

