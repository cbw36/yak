cmake_minimum_required(VERSION 3.10.0)
project(yak VERSION 0.2.0 LANGUAGES CXX CUDA)

# Default to C++14 for cross-compatibility
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra)
endif()

MACRO(CUDA_COMPUTE_TARGET_FLAGS arch_bin arch_ptx cuda_nvcc_target_flags)
    string(REGEX REPLACE "\\." "" ARCH_BIN_WITHOUT_DOTS "${${arch_bin}}")
    string(REGEX REPLACE "\\." "" ARCH_PTX_WITHOUT_DOTS "${${arch_ptx}}")

    set(cuda_computer_target_flags_temp "")

    # Tell NVCC to add binaries for the specified GPUs
    string(REGEX MATCHALL "[0-9()]+" ARCH_LIST "${ARCH_BIN_WITHOUT_DOTS}")
    foreach(ARCH IN LISTS ARCH_LIST)
        if (ARCH MATCHES "([0-9]+)\\(([0-9]+)\\)")
            # User explicitly specified PTX for the concrete BIN
            set(cuda_computer_target_flags_temp ${cuda_computer_target_flags_temp} -gencode;arch=compute_${CMAKE_MATCH_2},code=sm_${CMAKE_MATCH_1};)
        else()
            # User didn't explicitly specify PTX for the concrete BIN, we assume PTX=BIN
            set(cuda_computer_target_flags_temp ${cuda_computer_target_flags_temp} -gencode;arch=compute_${ARCH},code=sm_${ARCH};)
        endif()
    endforeach()

    # Tell NVCC to add PTX intermediate code for the specified architectures
    string(REGEX MATCHALL "[0-9]+" ARCH_LIST "${ARCH_PTX_WITHOUT_DOTS}")
    foreach(ARCH IN LISTS ARCH_LIST)
        set(cuda_computer_target_flags_temp ${cuda_computer_target_flags_temp} -gencode;arch=compute_${ARCH},code=compute_${ARCH};)
    endforeach()

    set(${cuda_nvcc_target_flags} ${cuda_computer_target_flags_temp})
ENDMACRO()

MACRO(APPEND_TARGET_ARCH_FLAGS)
    set(cuda_nvcc_target_flags "")
    CUDA_COMPUTE_TARGET_FLAGS(CUDA_ARCH_BIN CUDA_ARCH_PTX cuda_nvcc_target_flags)
    if (cuda_nvcc_target_flags)
        message(STATUS "CUDA NVCC target flags: ${cuda_nvcc_target_flags}")
        list(APPEND CUDA_NVCC_FLAGS ${cuda_nvcc_target_flags})
    endif()
ENDMACRO()

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/Modules/")
message(STATUS "CMAKE MODULE PATH IS ${CMAKE_MODULE_PATH}")

# TODO: Use built-in CUDA language support from CMake 3.8+ (requires different flags)
#enable_language(CUDA)

find_package(OpenMP REQUIRED)

#set(CUDA_LIBRARIES PUBLIC ${CUDA_LIBRARIES})
#find_package(CUDA REQUIRED)

if (CUDA_VERSION_MAJOR GREATER 8)
    set(CUDA_ARCH_BIN "3.0 5.0 5.2 6.0 6.1" CACHE STRING "Specify 'real' GPU architectures to build binaries for, BIN(PTX) format is supported")
else()
    set(CUDA_ARCH_BIN "2.0 2.1 3.0 5.0 5.2 6.0 6.1" CACHE STRING "Specify 'real' GPU architectures to build binaries for, BIN(PTX) format is supported")
endif()
APPEND_TARGET_ARCH_FLAGS()

message(STATUS ${CUDA_NVCC_FLAGS})

find_package(OpenCV 3 REQUIRED COMPONENTS core highgui)
find_package(Eigen3 REQUIRED)
find_package(PCL 1.8 REQUIRED COMPONENTS common io geometry)  # need to use PCL 1.9 because there's a GCC flag problem when building with 1.8
set_directory_properties( PROPERTIES COMPILE_DEFINITIONS "" )

link_directories(${PCL_LIBRARY_DIRS})
add_definitions(${PCL_DEFINITIONS})

# Core CUDA Library for depth image processing
add_library(${PROJECT_NAME} SHARED
  src/kfusion/core.cpp
  src/kfusion/device_memory.cpp
  src/kfusion/imgproc.cpp
  src/kfusion/kinfu.cpp
  src/kfusion/precomp.cpp
  src/kfusion/projective_icp.cpp
  src/kfusion/tsdf_volume.cpp
  src/cuda/imgproc.cu
  src/cuda/proj_icp.cu
  src/cuda/tsdf_volume.cu)
target_include_directories(${PROJECT_NAME} PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
  ${PCL_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
#  ${CUDA_INCLUDE_DIRS}
)
target_link_libraries(${PROJECT_NAME} PUBLIC
  ${OpenCV_LIBRARIES}
  ${PCL_LIBRARIES}
  Eigen3::Eigen)
#list(APPEND PACKAGE_LIBRARIES ${PROJECT_NAME})

# Jmeyer - Create a new interface library that I want to be the front end for future processing. It should support
# a minimal interface of pushing and image with a pose guess into the server and integrating.
add_library(${PROJECT_NAME}_frontend SHARED
    src/yak_server.cpp
    src/kfusion/tsdf_container.cpp)
target_include_directories(${PROJECT_NAME}_frontend PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:include>"
  ${PCL_INCLUDE_DIRS})
target_include_directories(${PROJECT_NAME}_frontend SYSTEM PUBLIC
  ${PCL_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME}_frontend
  ${PROJECT_NAME}
  ${PCL_LIBRARIES}
  Eigen3::Eigen)
list(APPEND PACKAGE_LIBRARIES ${PROJECT_NAME}_frontend)

# Marching Cubes Meshing Impl
add_library(${PROJECT_NAME}_marching_cubes SHARED
  src/mc/marching_cubes.cpp
  src/mc/marching_cubes_tables.cpp)
target_include_directories(${PROJECT_NAME}_marching_cubes PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME}_marching_cubes PUBLIC
  ${PCL_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME}_marching_cubes
    PUBLIC
  ${PROJECT_NAME}
  OpenMP::OpenMP_CXX
  Eigen3::Eigen)
list(APPEND PACKAGE_LIBRARIES ${PROJECT_NAME}_marching_cubes)

# Jmeyer Marching Cubes
add_executable(marching_cubes_tests src/mc/marching_cubes_tests.cpp)
target_include_directories(marching_cubes_tests PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:include>")
target_include_directories(marching_cubes_tests SYSTEM PUBLIC
  ${PCL_INCLUDE_DIRS})
target_link_libraries(marching_cubes_tests
  ${PROJECT_NAME}_marching_cubes
  ${PCL_LIBRARIES}
  Eigen3::Eigen)

install(DIRECTORY include/${PROJECT_NAME}
  DESTINATION include
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
  PATTERN "*.svn" EXCLUDE)

install(FILES package.xml DESTINATION share/${PROJECT_NAME})

install(TARGETS ${PROJECT_NAME} ${PACKAGE_LIBRARIES}
  EXPORT ${PROJECT_NAME}-targets DESTINATION lib)
install(EXPORT ${PROJECT_NAME}-targets
  NAMESPACE yak:: DESTINATION lib/cmake/${PROJECT_NAME})

include(CMakePackageConfigHelpers)
configure_package_config_file(
  ${CMAKE_CURRENT_LIST_DIR}/cmake/${PROJECT_NAME}-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION lib/cmake/${PROJECT_NAME}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO)

write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION} COMPATIBILITY ExactVersion)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake"
  DESTINATION lib/cmake/${PROJECT_NAME})

export(EXPORT ${PROJECT_NAME}-targets FILE
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-targets.cmake)

