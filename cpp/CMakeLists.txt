#------------------------------------------------------------------------------
# Top level CMakeLists.txt file for DOLFIN
cmake_minimum_required(VERSION 3.10)

#------------------------------------------------------------------------------
# Set project name and version number

project(DOLFINX_MPC VERSION 2019.2.9.99)

#------------------------------------------------------------------------------
# Set CMake options, see `cmake --help-policy CMP000x`

cmake_policy(VERSION 3.10)
if (POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()
if (POLICY CMP0075)
  cmake_policy(SET CMP0075 NEW)
endif()

#------------------------------------------------------------------------------
# Use C++17
set(CMAKE_CXX_STANDARD 17)

# Require C++17
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Do not enable compler-specific extensions
set(CMAKE_CXX_EXTENSIONS OFF)

#------------------------------------------------------------------------------
# General configuration

# Make sure CMake uses the correct DOLFINConfig.cmake for tests and demos
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${CMAKE_CURRENT_BINARY_DIR}/dolfinx_mpc)

#------------------------------------------------------------------------------
# configurable options for how we want to build

include(FeatureSummary)

option(BUILD_SHARED_LIBS "Build DOLFINX_MPC with shared libraries." ON)
add_feature_info(BUILD_SHARED_LIBS BUILD_SHARED_LIBS "Build DOLFINX_MPC with shared libraries.")

# Check for required package Eigen3
find_package(Eigen3 3.2.90 REQUIRED)
set_package_properties(Eigen3 PROPERTIES TYPE REQUIRED
  DESCRIPTION "Lightweight C++ template library for linear algebra"
  URL "http://eigen.tuxfamily.org")

# Check for required package UFC (part of ffcx)
MESSAGE(STATUS "Asking Python module FFCX for location of UFC...")
  find_package(PythonInterp 3 REQUIRED)
  execute_process(
	  COMMAND ${PYTHON_EXECUTABLE} -c "import ffcx.codegeneration, sys; sys.stdout.write(ffcx.codegeneration.get_include_path())"
    OUTPUT_VARIABLE UFC_INCLUDE_DIR
    )

  if (UFC_INCLUDE_DIR)
    set(UFC_INCLUDE_DIRS ${UFC_INCLUDE_DIR} CACHE STRING "Where to find ufc.h and ufc_geometry.h")

    execute_process(
      COMMAND ${PYTHON_EXECUTABLE} -c "import ffcx, sys; sys.stdout.write(ffcx.__version__)"
      OUTPUT_VARIABLE UFC_VERSION
      )

    if (UFC_FIND_VERSION)
      # Check if version found is >= required version
      if (NOT "${UFC_VERSION}" VERSION_LESS "${UFC_FIND_VERSION}")
        set(UFC_VERSION_OK TRUE)
      endif()
    else()
      # No specific version requested
      set(UFC_VERSION_OK TRUE)
    endif()
  endif()

  execute_process(
    COMMAND ${PYTHON_EXECUTABLE} -c "import ffcx.codegeneration, sys; sys.stdout.write(ffcx.codegeneration.get_signature())"
    OUTPUT_VARIABLE UFC_SIGNATURE
  )

mark_as_advanced(UFC_VERSION UFC_INCLUDE_DIRS UFC_SIGNATURE UFC_VERSION_OK)
# Standard package handling
find_package_handle_standard_args(UFC
                                  "UFC could not be found."
                                  UFC_INCLUDE_DIRS
                                  UFC_VERSION
                                  UFC_VERSION_OK
                                  UFC_SIGNATURE)
set_package_properties(UFC PROPERTIES TYPE REQUIRED
  DESCRIPTION "Unified language for form-compilers (part of FFC-X)"
  URL "https://github.com/fenics/ffcx")

# Check for required package DOLFINX
find_package(DOLFINX REQUIRED)
set_package_properties(DOLFINX PROPERTIES TYPE REQUIRED
    DESCRIPTION "New generation Dynamic Object-oriented Library for - FINite element computation"
    URL "https://github.com/FEniCS/dolfinx"
    PURPOSE "Main dependency of library")

feature_summary(WHAT ALL)


#------------------------------------------------------------------------------
# Installation of DOLFIN_MPC library

add_library(dolfinx_mpc "")  # The "" is needed for older CMake. Remove later.

if (UFC_FOUND)
    target_include_directories(dolfinx_mpc PRIVATE ${UFC_INCLUDE_DIRS})
endif()
target_link_libraries(dolfinx_mpc PUBLIC dolfinx)


# Delcare the library (target)

#------------------------------------------------------------------------------
include(GNUInstallDirs)

#------------------------------------------------------------------------------

install(FILES dolfinx_mpc.h  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/dolfinx_mpc COMPONENT Development)
install(FILES MultiPointConstraint.h utils.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/dolfinx_mpc COMPONENT Development)

# Add source files to the target
target_sources(dolfinx_mpc PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/MultiPointConstraint.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/utils.cpp
)

# Set target include location (for build and installed)
target_include_directories(dolfinx_mpc PUBLIC
                           $<INSTALL_INTERFACE:include>
                           "$<BUILD_INTERFACE:${DOLFINX_MPC_SOURCE_DIR};${DOLFINX_MPC_SOURCE_DIR}/dolfinx_mpc>")

# Set target properties
set_target_properties(dolfinx_mpc PROPERTIES
  VERSION ${DOLFINX_MPC_VERSION}
  SOVERSION ${DOLFINX_MPC_VERSION_MAJOR}.${DOLFINX_MPC_VERSION_MINOR})


# Add version to definitions (public)
target_compile_definitions(dolfinx_mpc PUBLIC DOLFINX_MPC_VERSION="${DOLFINX_MPC_VERSION}")


#------------------------------------------------------------------------------
# Eigen3, MPI and PETSC
# See DOLFINX for explanation of this variable
set(DOLFINX_EIGEN_MAX_ALIGN_BYTES "32" CACHE STRING "\
Minimum alignment in bytes used for Eigen data structures. Set to 32 for \
compatibility with AVX user-compiled code and 64 for AVX-512 user-compiled \
code. Set to 0 for ideal alignment according to -march. Note that if an architecture \
flag (e.g. \"-march=skylake-avx512\") is set for DOLFIN, Eigen will use the \
appropriate ideal alignment instead if it is stricter. Otherwise, the value \
of this variable will be used by Eigen for the alignment of all data structures.\\
")

target_include_directories(dolfinx_mpc SYSTEM PUBLIC ${EIGEN3_INCLUDE_DIR})
target_compile_definitions(dolfinx_mpc PUBLIC "EIGEN_MAX_ALIGN_BYTES=${DOLFINX_EIGEN_MAX_ALIGN_BYTES}")

# MPI
target_link_libraries(dolfinx_mpc PUBLIC MPI::MPI_CXX)

# petsc
target_link_libraries(dolfinx_mpc PUBLIC PETSC::petsc)
target_link_libraries(dolfinx_mpc PRIVATE PETSC::petsc_static)

#------------------------------------------------------------------------------
# Install dolfinx library and header files
install(TARGETS dolfinx_mpc
  EXPORT DOLFINX_MPCTargets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT RuntimeExecutables
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT RuntimeLibraries
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT Development
  )

# Generate DOLFINTargets.cmake
install(EXPORT DOLFINX_MPCTargets DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/dolfinx_mpc)

include(CMakePackageConfigHelpers)

configure_package_config_file(${DOLFINX_MPC_SOURCE_DIR}/DOLFINX_MPCConfig.cmake.in
${CMAKE_BINARY_DIR}/dolfinx_mpc/DOLFINX_MPCConfig.cmake
INSTALL_DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/dolfinx_mpc)

# Install CMake helper files
install(
  FILES
  ${CMAKE_BINARY_DIR}/dolfinx_mpc/DOLFINX_MPCConfig.cmake
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/dolfinx_mpc
  COMPONENT Development)

#------------------------------------------------------------------------------
