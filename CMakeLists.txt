#
# Copyright (c) 2021-2022 INRIA
#

cmake_minimum_required(VERSION 3.1)

set(PROJECT_NAME pycppad)
set(PROJECT_DESCRIPTION
    "Python bindings for CppAD and CppADCodeGen using Boost.Python")
set(PROJECT_URL "https://github.com/simple-robotics/pycppad")
set(PROJECT_CUSTOM_HEADER_EXTENSION "hpp")
set(PROJECT_USE_CMAKE_EXPORT TRUE)

# Project configuration
set(PROJECT_USE_CMAKE_EXPORT TRUE)
set(CMAKE_VERBOSE_MAKEFILE TRUE)
set(CXX_DISABLE_WERROR TRUE)

# Check if the submodule cmake have been initialized
if(NOT EXISTS "${CMAKE_SOURCE_DIR}/cmake/base.cmake")
  message(
    FATAL_ERROR
      "\nPlease run the following command first:\ngit submodule update --init\n"
  )
endif()

include(cmake/base.cmake)
include(cmake/boost.cmake)
include(cmake/python.cmake)
include(cmake/ide.cmake)

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/find-external/CppAD/"
                      ${CMAKE_MODULE_PATH})

# Project definition
compute_project_args(PROJECT_ARGS LANGUAGES CXX)
project(${PROJECT_NAME} ${PROJECT_ARGS})
check_minimal_cxx_standard(11 ENFORCE)

# Project options
option(SUFFIX_SO_VERSION "Suffix library name with its version" ON)
option(BUILD_WITH_CPPAD_CODEGEN_BINDINGS
       "Build the python bindings for code generation (via CppADCodeGen)" OFF)

findpython()

set_boost_default_options()
add_project_dependency(Boost REQUIRED)
search_for_boost_python(REQUIRED)
export_boost_default_options()

if(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)
  add_project_dependency(
    cppadcg 2.4.1 REQUIRED PKG_CONFIG_REQUIRES "cppadcg >= 2.4.1"
  )# CppADCodeGen 2.4.1 is the first version to check the minimal version of
   # CppAD
  add_definitions(-DPYCPPAD_WITH_CPPAD_CODEGEN_BINDINGS)
endif(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)

add_project_dependency(cppad 20180000.0 REQUIRED PKG_CONFIG_REQUIRES
                       "cppad >= 20180000.0")
add_project_dependency(Eigen3 REQUIRED PKG_CONFIG_REQUIRES "eigen3 >= 3.0.5")
add_project_dependency(eigenpy 2.7.6 REQUIRED)

set(${PROJECT_NAME}_HEADERS
    include/${PROJECT_NAME}/fwd.hpp
    include/${PROJECT_NAME}/ad.hpp
    include/${PROJECT_NAME}/independent.hpp
    include/${PROJECT_NAME}/ad_fun.hpp
    include/${PROJECT_NAME}/cast.hpp
    include/${PROJECT_NAME}/cppad.hpp
    include/${PROJECT_NAME}/cppad-scalar.hpp
    include/${PROJECT_NAME}/utils/scope.hpp
    # Generated headers
    ${${PROJECT_NAME}_BINARY_DIR}/include/${PROJECT_NAME}/config.hpp
    ${${PROJECT_NAME}_BINARY_DIR}/include/${PROJECT_NAME}/deprecated.hpp
    ${${PROJECT_NAME}_BINARY_DIR}/include/${PROJECT_NAME}/warning.hpp)

set(${PROJECT_NAME}_SOURCES src/cppad.cpp)

if(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)
  list(
    APPEND
    ${PROJECT_NAME}_HEADERS
    include/${PROJECT_NAME}/codegen/cg.hpp
    include/${PROJECT_NAME}/codegen/cppadcg-scalar.hpp
    include/${PROJECT_NAME}/codegen/code-handler.hpp
    include/${PROJECT_NAME}/codegen/lang/c/language-c.hpp
    include/${PROJECT_NAME}/codegen/lang/c/lang-c-default-var-name-gen.hpp)
endif(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)

add_library(${PROJECT_NAME} SHARED ${${PROJECT_NAME}_SOURCES}
                                   ${${PROJECT_NAME}_HEADERS})

if(SUFFIX_SO_VERSION)
  set_target_properties(${PROJECT_NAME} PROPERTIES SOVERSION ${PROJECT_VERSION})
endif(SUFFIX_SO_VERSION)

target_link_boost_python(${PROJECT_NAME} PUBLIC)
target_link_libraries(${PROJECT_NAME} PUBLIC ${cppad_LIBRARY} eigenpy::eigenpy)
if(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)
  target_include_directories(${PROJECT_NAME} SYSTEM
                             PUBLIC ${cppadcg_INCLUDE_DIR})
  target_link_libraries(${PROJECT_NAME} PUBLIC ${CMAKE_DL_LIBS})
endif(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)

if(MSVC)
  target_compile_options(${PROJECT_NAME} PUBLIC "/bigobj")
endif()

# Do not report: -Wconversion as the BOOST_PYTHON_FUNCTION_OVERLOADS implicitly
# converts. -Wcomment as latex equations have multi-line comments.
if(NOT WIN32)
  target_compile_options(${PROJECT_NAME} PRIVATE -Wno-conversion -Wno-comment)
endif(NOT WIN32)
if(WIN32)
  target_compile_definitions(${PROJECT_NAME} PRIVATE -DNOMINMAX)
  target_link_libraries(${PROJECT_NAME} PUBLIC ${PYTHON_LIBRARY})
endif(WIN32)

target_include_directories(
  ${PROJECT_NAME} SYSTEM
  PUBLIC ${Boost_INCLUDE_DIRS} ${EIGEN3_INCLUDE_DIR} ${PYTHON_INCLUDE_DIRS}
         $<INSTALL_INTERFACE:include>)
target_include_directories(${PROJECT_NAME} PUBLIC $<INSTALL_INTERFACE:include>)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${TARGETS_EXPORT_NAME}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR}
  INCLUDES
  DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_FULL_BINDIR})

add_header_group(${PROJECT_NAME}_HEADERS)
add_source_group(${PROJECT_NAME}_SOURCES)

add_subdirectory(python)
add_subdirectory(example)

pkg_config_append_libs(${PROJECT_NAME})
pkg_config_append_cflags("-I${PYTHON_INCLUDE_DIRS}")
pkg_config_append_boost_libs(${BOOST_COMPONENTS})

# Install catkin package.xml
install(FILES package.xml DESTINATION share/${PROJECT_NAME})
