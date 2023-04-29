#
# Copyright (c) 2021 INRIA
#

# --- LIBRARY --- #
set(PYWRAP ${PROJECT_NAME}_pywrap)
set(PYWRAP
    ${PYWRAP}
    PARENT_SCOPE)

make_directory("${${PROJECT_NAME}_BINARY_DIR}/python/${PROJECT_NAME}")

add_custom_target(python)
set_target_properties(python PROPERTIES EXCLUDE_FROM_DEFAULT_BUILD True)

add_library(${PYWRAP} SHARED main.cpp)
add_dependencies(python ${PYWRAP})
target_link_libraries(${PYWRAP} PUBLIC ${PROJECT_NAME})
target_link_boost_python(${PYWRAP} PUBLIC)
# BOOST_PYTHON_MEMBER_FUNCTION_OVERLOADS spews conversion warnings from int to
# long unsigned int. Unfortunately, using literals does not work in a macro. As
# such, this turns them off for the entire wrapper:
if(NOT WIN32)
  target_compile_options(${PYWRAP} PRIVATE "-Wno-conversion")
endif()
if(IS_ABSOLUTE ${PYTHON_SITELIB})
  set(${PYWRAP}_INSTALL_DIR ${PYTHON_SITELIB}/${PROJECT_NAME})
else()
  set(${PYWRAP}_INSTALL_DIR
      ${CMAKE_INSTALL_PREFIX}/${PYTHON_SITELIB}/${PROJECT_NAME})
endif()
if(WIN32)
  target_compile_definitions(${PYWRAP} PRIVATE -DNOMINMAX)
  target_link_libraries(${PYWRAP} PUBLIC ${PYTHON_LIBRARY})
endif(WIN32)

set_target_properties(
  ${PYWRAP}
  PROPERTIES PREFIX ""
             SUFFIX ${PYTHON_EXT_SUFFIX}
             OUTPUT_NAME "${PYWRAP}"
             LIBRARY_OUTPUT_DIRECTORY
             "${CMAKE_BINARY_DIR}/python/${PROJECT_NAME}")

if(UNIX AND NOT APPLE)
  set_target_properties(${PYWRAP} PROPERTIES INSTALL_RPATH "\$ORIGIN/../../..")
endif()

install(TARGETS ${PYWRAP} DESTINATION ${${PYWRAP}_INSTALL_DIR})

# --- INSTALL SCRIPTS
set(PYTHON_FILES __init__.py)

foreach(python ${PYTHON_FILES})
  python_build(${PROJECT_NAME} ${python})
  install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/pycppad/${python}"
          DESTINATION ${${PYWRAP}_INSTALL_DIR})
endforeach(python)
