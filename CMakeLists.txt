cmake_minimum_required(VERSION 3.5)

set(_policy_list
    CMP0012
    CMP0023
    CMP0028
    CMP0042
    CMP0048
    CMP0051
    CMP0054
    CMP0057
    CMP0074
    CMP0076)
foreach(_policy ${_policy_list})
  if(POLICY ${_policy})
    cmake_policy(SET ${_policy} NEW)
  endif()
  # CMP0012: if() recognizes numbers and booleans
  # CMP0028: :: in target names
  # CMP0042: MACOS_RPATH
  # CMP0048: allow VERSION in project()
  # CMP0051: list TARGET_OBJECTS in SOURCES property
  # CMP0054: no more de-referencing of "expr" in if() statements
  # CMP0057: if IN_LIST
  # CMP0074: XXX_ROOT variables for find_package(XXX)
  # CMP0076: target_sources relative paths
endforeach()

if(CMAKE_VERSION VERSION_LESS 3.11)
  # Test target was always reserved prior to CMake 3.11
  cmake_policy(SET CMP0037 OLD)
endif()

# ==============================================================================
# Macro definitions

include(${CMAKE_CURRENT_LIST_DIR}/cmake/macros.cmake)

# ==============================================================================
# Create the HiQSimulator project

set(CMAKE_OSX_SYSROOT "") # Prevent CMake from adding the -isysroot options
project(HiQSimulator LANGUAGES CXX)

# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE
               PROPERTY STRINGS
                        "Debug"
                        "Release"
                        "RelWithDebInfo")
endif()

list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_CURRENT_LIST_DIR}/cmake)

#==============================================================================
# Options

include(${CMAKE_CURRENT_LIST_DIR}/cmake/options.cmake)

# ==============================================================================
# Package dependencies

include(${CMAKE_CURRENT_LIST_DIR}/cmake/packages.cmake)

#==============================================================================
# Setup compiler flags

set(CMAKE_CXX_STANDARD 14) # might decay to C++11
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

check_compiler_flags(_compile_flags_release
                     "-ffast-math /fp:fast -fast"
                     "-O3 /Ox"
                     "-march=native -xHost /QxHost")

add_compile_options(
  "$<$<AND:$<CONFIG:RELEASE>,$<COMPILE_LANGUAGE:CXX>>:${_compile_flags_release_cxx}>"
  )

check_compiler_flags(_avx2_flag "-mavx2 -xCORE-AVX2 /QxCORE-AVX2 /arch:AVX2")

# ------------------------------------------------------------------------------

set(FLAGS_MPI)
set(CMAKE_CXX_FLAGS_DEBUG_MPI
    "${CMAKE_CXX_FLAGS_DEBUG} ${MPI_COMPILE_FLAGS} ${FLAGS_MPI}")
set(CMAKE_CXX_FLAGS_RELEASE_MPI
    "${CMAKE_CXX_FLAGS_RELEASE} ${MPI_COMPILE_FLAGS} ${FLAGS_MPI}")

set(CMAKE_LINK_FLAGS_DEBUG_MPI "${MPI_LINK_FLAGS}")
set(CMAKE_LINK_FLAGS_RELEASE_MPI "${MPI_LINK_FLAGS}")

# ------------------------------------------------------------------------------

set(_doc_targets)

# ==============================================================================

add_subdirectory(pybind11)

set(SRC_DIR ${CMAKE_CURRENT_LIST_DIR}/src)
add_subdirectory(${SRC_DIR})
include_directories(${SRC_DIR})

# ==============================================================================

add_executable(qswap qswap.cpp)
add_object_library_dependency(qswap PUBLIC permutations)
target_link_libraries(qswap PUBLIC Boost::boost Boost::program_options)

# --------------------------------------

add_executable(qswap-mpi qswap-mpi.cpp src/simulator-mpi/SwapperMT.cpp)
add_object_library_dependency(qswap-mpi PUBLIC permutations)
target_compile_definitions(qswap-mpi PRIVATE -DSIMULATOR_LIBRARY_EXPORT)
target_link_libraries(qswap-mpi
                      PUBLIC ${MPI_LIBRARIES}
                             Boost::program_options
                             Boost::mpi
                             Boost::serialization
                             Boost::thread
                             ${OpenMP_tgt})

# ------------------------------------------------------------------------------

add_executable(opyt opyt.cpp)
target_link_libraries(opyt
                      Boost::program_options
                      Boost::mpi
                      Boost::serialization
                      Boost::thread
                      ${OpenMP_tgt}
                      glog::glog)

# --------------------------------------

add_executable(opyt-mpi opyt-mpi.cpp)
target_link_libraries(opyt-mpi
                      ${MPI_LIBRARIES}
                      Boost::program_options
                      Boost::mpi
                      Boost::serialization
                      Boost::thread
                      ${OpenMP_tgt}
                      glog::glog)

# ------------------------------------------------------------------------------

add_executable(socket-test socket-test.cpp)
target_link_libraries(socket-test
                      Boost::program_options
                      Boost::mpi
                      Boost::serialization
                      Boost::thread
                      ${OpenMP_tgt})

# ------------------------------------------------------------------------------

add_executable(himembind himembind.cpp)
target_link_libraries(himembind hwloc::hwloc)

# ------------------------------------------------------------------------------

pybind11_add_module(_sched_cpp
                    _sched_cpp.cpp
                    src/scheduler/swap_scheduler.cpp
                    src/scheduler/cluster_scheduler.cpp
                    src/scheduler/convertors.cpp)
target_sources(_sched_cpp
               INTERFACE
               ${SRC_DIR}/scheduler/swap_scheduler.h
               ${SRC_DIR}/scheduler/cluster_scheduler.h
               ${SRC_DIR}/scheduler/definitions.h
               ${SRC_DIR}/scheduler/convertors.h)
target_link_libraries(_sched_cpp PUBLIC glog::glog)
list(APPEND _doc_targets _sched_cpp)

# ------------------------------------------------------------------------------

define_object_library(SimulatorMPI_o
                      SOURCES
                      ${SRC_DIR}/simulator-mpi/SimulatorMPI.cpp
                      ${SRC_DIR}/simulator-mpi/SwapperMT.cpp
                      HEADERS
                      ${SRC_DIR}/simulator-mpi/swapping.hpp
                      ${SRC_DIR}/simulator-mpi/fusion_mpi.hpp
                      ${SRC_DIR}/simulator-mpi/alignedallocator.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernel1.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernel2.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernel3.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernel4.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernel5.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernels.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/nointrin/kernels_diag.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/cintrin.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernel1.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernel2.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernel3.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernel4.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernel4_cf.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernel5.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernels.hpp
                      ${SRC_DIR}/simulator-mpi/kernels/intrin/kernels_diag.hpp
                      ${SRC_DIR}/simulator-mpi/SimulatorMPI.hpp
                      ${SRC_DIR}/simulator-mpi/SwapperMT.hpp
                      DEPENDENCIES
                      Boost::boost)
if(WIN32)
  target_compile_definitions(SimulatorMPI_o PRIVATE -DSIMULATOR_LIBRARY_EXPORT)
endif()

add_library(SimulatorMPI SHARED ${SRC_DIR}/simulator-mpi/dummy.cpp)
add_object_library_dependency(SimulatorMPI PUBLIC SimulatorMPI_o)
add_object_library_dependency(SimulatorMPI PUBLIC permutations)
add_avx2_to_target(SimulatorMPI)
target_link_libraries(SimulatorMPI
                      PUBLIC ${MPI_LIBRARIES}
                             Boost::program_options
                             Boost::mpi
                             Boost::serialization
                             Boost::thread
                             ${OpenMP_tgt}
                             glog::glog)

# ------------------------------------------------------------------------------

add_executable(cppsim-mpi cppsim-mpi.cpp)
add_avx2_to_target(cppsim-mpi)
target_link_libraries(cppsim-mpi
                      PUBLIC SimulatorMPI
                             ${MPI_LIBRARIES}
                             Boost::program_options
                             Boost::mpi
                             Boost::serialization
                             Boost::thread
                             glog::glog
                             ${OpenMP_tgt})

# ------------------------------------------------------------------------------

pybind11_add_module(_cppsim_mpi _cppsim_mpi.cpp)
add_object_library_dependency(_cppsim_mpi PUBLIC SimulatorMPI_o)
add_avx2_to_target(_cppsim_mpi)
target_link_libraries(_cppsim_mpi
                      PUBLIC ${MPI_LIBRARIES}
                             Boost::mpi
                             Boost::serialization
                             Boost::thread
                             glog::glog
                             ${OpenMP_tgt})

# ------------------------------------------------------------------------------

pybind11_add_module(_cppstabsim _cppstabsim.cpp)
add_avx2_to_target(_cppstabsim)
target_link_libraries(_cppstabsim PRIVATE Boost::boost ${xsimd_tgt})

# ==============================================================================
# Unit testing

if(BUILD_TESTING)
  include(CTest)
  include(${CMAKE_CURRENT_LIST_DIR}/src/test/CMakeLists.txt)
  add_custom_target(build_all_test DEPENDS "${_boost_test_list}")
else()
  # Essentially for Travis.CI builds in case the Boost version is too old
  add_custom_target(build_all_test)
  add_custom_target(test)
endif()

# ==============================================================================

find_package(Doxygen)
if(Doxygen_FOUND)
  add_subdirectory(docs)
endif()
