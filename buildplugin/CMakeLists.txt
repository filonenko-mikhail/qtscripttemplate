cmake_minimum_required(VERSION 2.8)

# Variable for project name, used in many places
#set (PROJECT YOUR_SCRIPT_PLUGIN_NAME)
#set YOUR PACKAGE NAME, it used by QtScriptEngine::importExtension() method
#set (QS_PACKAGE_NAME YOUR_PACKAGE_NAME)

# Declare project
project(${PROJECT})

find_package(Qt4 REQUIRED)

# PROJECT NAME
string(TOUPPER ${PROJECT_NAME} PROJECT_TO_UPPER)

message("Building project ${PROJECT_NAME}")

message("
Configuration
-------------
CMAKE_BUILD_TYPE        = ${CMAKE_BUILD_TYPE}
CXXFLAGS                = $ENV{CXXFLAGS}
CMAKE_CXX_FLAGS         = ${CMAKE_CXX_FLAGS}
LDFLAGS                 = $ENV{LDFLAGS}
CMAKE_EXE_LINKER_FLAGS  = ${CMAKE_EXE_LINKER_FLAGS}
CMAKE_INSTALL_PREFIX    = ${CMAKE_INSTALL_PREFIX}
")

# SOURCE FILES AND DIRECTORIES

if (NOT INCLUDE_DIR)
  get_filename_component(INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../include" REALPATH)
  message(WARNING "INCLUDE_DIR is not defined. Using default: \"${INCLUDE_DIR}\"")
endif(NOT INCLUDE_DIR)

if (NOT LIBRARIES)
  get_filename_component(LIBRARY_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../lib" REALPATH)
	set(LIBRARIES_WARN TRUE)
  file(GLOB_RECURSE FILES_LIST "${LIBRARY_PATH}/*")
  foreach(FILE ${FILES_LIST})
    get_filename_component(IS_LIBRARY ${FILE} NAME_WE)
    string(REGEX REPLACE "^lib" "" IS_LIBRARY ${IS_LIBRARY})
    find_library(ISIT_LIBRARY NAMES ${IS_LIBRARY} PATHS ${LIBRARY_PATH} NO_DEFAULT_PATH)
    if (ISIT_LIBRARY)
      list(APPEND LIBRARIES ${ISIT_LIBRARY})
    endif(ISIT_LIBRARY)
  endforeach(FILE)
endif(NOT LIBRARIES)
list(REMOVE_DUPLICATES LIBRARIES)
if (LIBRARIES_WARN)
	message(WARNING "LIBRARIES is not defined. Using all libraries in \"${LIBRARY_PATH}\":\n${LIBRARIES}")
endif (LIBRARIES_WARN)

include_directories(${INCLUDE_DIR})

# QT CUSTOMIZATION
# For example
# using qt script
set(QT_USE_QTSCRIPT TRUE)
# SET DONT USING Qt MODULES
# set(QT_DONT_USE_QTGUI TRUE)
# SET USING Qt MODULES
#set(QT_USE_QTSQL TRUE)

# Create libraries list
include(${QT_USE_FILE})


# -----------------------------------------------------------------------------------------------------
# GENERATING C++ CLASSES-WRAPPERS FOR QT SCRIPT
# Qt Script Bindings Generator options
string(REPLACE "." "_" QS_PACKAGE_NAME_UND ${QS_PACKAGE_NAME})

set (QS_GENERATED_PATH "${CMAKE_CURRENT_BINARY_DIR}/generated_cpp/${QS_PACKAGE_NAME_UND}")

set (QS_BUILD_IN "${CMAKE_CURRENT_SOURCE_DIR}/plugin_build.txt.in")
set (QS_BUILD "${CMAKE_CURRENT_BINARY_DIR}/plugin_build.txt")

set (QS_TYPESYSTEM_IN "${CMAKE_CURRENT_SOURCE_DIR}/plugin_typesystem.xml.in")
set (QS_TYPESYSTEM "${CMAKE_CURRENT_BINARY_DIR}/plugin_typesystem.xml")

set (QS_MASTER_H "${CMAKE_CURRENT_BINARY_DIR}/master_include.h")

# before compilation we must generate plugin source files by qt script bindings generator
# exec string like this, where QSBG_DIR environment variable contains path to qscript bindings generator source directory
# $QSBG_DIR/generator/generator --output-directory=. --include-paths=$QTDIR/include
configure_file(
  ${QS_BUILD_IN}
  ${QS_BUILD}
  IMMEDIATE @ONLY)

configure_file(
  ${QS_TYPESYSTEM_IN}
  ${QS_TYPESYSTEM}
  IMMEDIATE @ONLY)

set(QSBG_DIR $ENV{QSBG_DIR})
if(NOT QSBG_DIR)
  get_filename_component(QSBG_DIR "../../qtscriptgenerator" ABSOLUTE)
  message(WARNING " Please set environment variable \"QSBG_DIR\", which must contains path to qt script bindings generator project directory. Used default: \"${QSBG_DIR}\"")
endif(NOT QSBG_DIR)

# Find qt script bindings generator
find_program(QSBG_EXECUTABLE generator PATHS ${QSBG_DIR}/generator ${QSBG_DIR}/generator/debug ${QSBG_DIR}/generator/release NO_DEFAULT_PATH)

# If plugin.cpp older than current CMakeLists.txt, than regenerate plugin sources by qt script bindings generator
if (UNIX)
  execute_process(COMMAND find ${INCLUDE_DIR} -newer ${QS_GENERATED_PATH}/plugin.cpp -regex ".*\\.h"
    RESULT_VARIABLE find_result1
    OUTPUT_VARIABLE find_outval1)
  execute_process(COMMAND find ${CMAKE_CURRENT_SOURCE_DIR} -newer ${QS_GENERATED_PATH}/plugin.cpp -regex ".*_typesystem\\.xml\\.in"
    RESULT_VARIABLE find_result2
    OUTPUT_VARIABLE find_outval2)
  if (NOT "${find_result1}" STREQUAL "0")
    set(GENERATE_CPP_WRAPPERS TRUE)
  endif (NOT "${find_result1}" STREQUAL "0")
  if (NOT "${find_result2}" STREQUAL "0")
    set(GENERATE_CPP_WRAPPERS TRUE)
  endif (NOT "${find_result2}" STREQUAL "0")
  if(find_outval1 OR find_outval2)
    set(GENERATE_CPP_WRAPPERS TRUE)
  endif(find_outval1 OR find_outval2)
else(UNIX)
  set(GENERATE_CPP_WRAPPERS TRUE)
endif (UNIX)

if (QSBG_EXECUTABLE AND GENERATE_CPP_WRAPPERS)
  ## DANGER Remove old master.h file
  file(REMOVE ${QS_MASTER_H})
  file(GLOB_RECURSE INCLUDED_HEADERS  "${INCLUDE_DIR}/*.h")
  #message(STATUS "status 	file(GLOB_RECURSE INCLUDED_HEADERS ${INCLUDE_DIR} "*.h")")
  foreach(HEADER_FILE ${INCLUDED_HEADERS})
    file(APPEND ${QS_MASTER_H} "#include \"${HEADER_FILE}\"\n")
  endforeach(HEADER_FILE)
  ## DANGER Remove directory with old generated sources
  #file(REMOVE_RECURSE ${QS_GENERATED_PATH})
  # Walk throw file list to

  message("Qt script bindings generator executable founded")
  message("Generating c++ wrapper classes:")
  message("${QSBG_EXECUTABLE}"
		"${QS_MASTER_H}"
		"${QS_BUILD}"
		"--output-directory=${CMAKE_CURRENT_BINARY_DIR}")
  ## Execute qt script bindings generator executable
  execute_process(COMMAND ${QSBG_EXECUTABLE}
    ${QS_MASTER_H}
    ${QS_BUILD}
    --output-directory=${CMAKE_CURRENT_BINARY_DIR}
    WORKING_DIRECTORY ${QSBG_DIR}/generator
    RESULT_VARIABLE qsbg_retval
    OUTPUT_VARIABLE gsbg_outval
    ERROR_VARIABLE qsbg_errval)
  if (NOT "${qsbg_retval}" STREQUAL 0)
    message(FATAL_ERROR "Problem when generating c++ wrapper classes:"
			"${qsbg_errval}"
			"${qsbg_outval}"
			"${qsbg_retval}")
  else (NOT "${qsbg_retval}" STREQUAL 0)
    message("Generating output:"
			"${qsbg_errval}"
			"${qsbg_outval}")
    if(UNIX)
      execute_process(COMMAND touch ${QS_GENERATED_PATH}/plugin.cpp
        OUTPUT_VARIABLE touch_outval)
    endif(UNIX)
  endif (NOT "${qsbg_retval}" STREQUAL 0)
endif (QSBG_EXECUTABLE AND GENERATE_CPP_WRAPPERS)

include("${QS_GENERATED_PATH}/${QS_PACKAGE_NAME_UND}.cmake")
list(APPEND QS_SOURCES "${QS_GENERATED_PATH}/plugin.cpp")
# Include sources generated by Qt Script Bindings Generator
include_directories(${QS_GENERATED_PATH})

# END CPP CLASSES-WRAPPERS FOR QT SCRIPT (OUTPUT SOURCES VARIABLE: QS_SOURCES)
# -----------------------------------------------------------------------------------------------------

# PLUGIN COMPILATION
add_definitions(-DQT_PLUGIN)
add_definitions(-DQT_NO_DEBUG)

add_library(${PROJECT_NAME} SHARED
	${QS_HEADERS}
  ${QS_SOURCES})

set_target_properties(${PROJECT_NAME} PROPERTIES 
	LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/../plugins/script"
	RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/../plugins/script")

# LINKING
# main library
target_link_libraries(${PROJECT_NAME} ${QT_LIBRARIES}
  ${LIBRARIES})

install(TARGETS ${PROJECT_NAME} DESTINATION ${QT_PLUGINS_DIR}/script)
install(DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/doc" DESTINATION ${QT_DOC_DIR}/${QS_PACKAGE_NAME_UND} COMPONENT "Documentation" PATTERN "*.htm?")