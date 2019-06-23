#=========================================================================================
# Copyright (c) 2013, Christian Gehring, Hannes Sommer, Paul Furgale, Remo Diethelm
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#     * Redistributions of source code must retain the above copyright
#       notice, this list of conditions and the following disclaimer.
#     * Redistributions in binary form must reproduce the above copyright
#       notice, this list of conditions and the following disclaimer in the
#       documentation and/or other materials provided with the distribution.
#     * Neither the name of the Autonomous Systems Lab, ETH Zurich nor the
#       names of its contributors may be used to endorse or promote products
#       derived from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
# ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
# WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL Christian Gehring, Hannes Sommer, Paul Furgale,
# Remo Diethelm BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY,
# OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE
# GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
# HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
# SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#=========================================================================================
cmake_minimum_required(VERSION 3.10)
project(kindr VERSION 1.0.1 LANGUAGES CXX)

# CMake helper modules
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

#==
# Options
#==

option(KINDR_DOC "Build Kindr documentation." OFF)

#==
# Dependencies
#==

# Add the helper CMake scripts for finding dependencies
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake" ${CMAKE_MODULE_PATH})

# Eigen: A C++ template library for linear algebra
find_package(Eigen REQUIRED)
if(EIGEN_FOUND)
  message(STATUS "Eigen:")
  message(STATUS "  Version: ${EIGEN_VERSION}")
  message(STATUS "  Includes: ${EIGEN_INCLUDE_DIRS}")
endif()

#==
# Build configuration
#==

# Noisily default to Release build
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Don't build tests if not specified
if(NOT BUILD_TEST)
  message(STATUS "Setting build-tests to false as not specified.")
  set(BUILD_TEST false CACHE BOOL "Choose whether to build tests." FORCE)
  set_property(CACHE BUILD_TEST PROPERTY STRINGS
    "True" "False")
endif()

# Conditionally build gtest from source
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/gtest/" AND BUILD_TEST)
  message(STATUS "Building GTest!")
  option(BUILD_GTEST "Build GTest from source." ON)
  add_subdirectory(gtest gtest)
  enable_testing()
endif()

#==
# Build target libraries
#==

# The header-only library target
add_library(${PROJECT_NAME} INTERFACE)
target_include_directories(${PROJECT_NAME}
  INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
    $<BUILD_INTERFACE:${EIGEN_INCLUDE_DIRS}>
    $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/include>
)
target_link_libraries(${PROJECT_NAME}
  INTERFACE
    pthread
)
target_compile_features(${PROJECT_NAME} INTERFACE cxx_std_11)
target_compile_options(${PROJECT_NAME} INTERFACE "-Wall" "-Wextra" "-Wl,--no-as-needed" ${KINDR_COMPILE_OPTS})

# Alias the common headers target within the joint kindr project namespace
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

#==
# Package targets
#==

install(TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}-targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

export(TARGETS ${PROJECT_NAME}
  NAMESPACE kindr::
  FILE ${PROJECT_NAME}-targets.cmake
)

install(EXPORT ${PROJECT_NAME}-targets
  FILE ${PROJECT_NAME}-targets.cmake
  NAMESPACE kindr::
  DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
)

#==
# Package configuration
#==

configure_package_config_file(
  ${CMAKE_CURRENT_LIST_DIR}/cmake/${PROJECT_NAME}-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
)

write_basic_package_version_file(cmake/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

#==
# Package installation
#==

install(
  FILES
    ${CMAKE_CURRENT_BINARY_DIR}/cmake/${PROJECT_NAME}-config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/cmake/${PROJECT_NAME}-config-version.cmake
  DESTINATION
    ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
)

install(
  DIRECTORY
    ${CMAKE_CURRENT_LIST_DIR}/include/${PROJECT_NAME}
  DESTINATION
    ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
  FILES
    ${CMAKE_CURRENT_LIST_DIR}/package.xml
  DESTINATION
    ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}
)

#==
# Uninstallation
#==

configure_file(
  "${CMAKE_CURRENT_LIST_DIR}/cmake/kindr-uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/kindr-uninstall.cmake"
  IMMEDIATE @ONLY
)

if (NOT TARGET uninstall)
  add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake/kindr-uninstall.cmake)
endif()

#==
# Build unit tests
#==

# Add unit tests
if(BUILD_GTEST)
  add_subdirectory(test)
endif()

# Add Doxygen documentation
if(KINDR_DOC)
  add_subdirectory(doc/doxygen)
endif()

# EOF

