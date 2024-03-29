cmake_minimum_required (VERSION 2.6)

enable_testing()

# Only set project name if OpenMesh is built as stand-alone library
if("${PROJECT_NAME}" STREQUAL "")
    project (OpenMesh)
endif()

if (NOT WIN32 AND NOT CMAKE_BUILD_TYPE)
  message(STATUS "No build type selected, default to Release")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

# add our macro directory to cmake search path
set (CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
set (CMAKE_DEBUG_POSTFIX "d")

# include our cmake files
include (ACGCommon)

acg_get_version ()


# Disable package building when built as an external library
if(${PROJECT_NAME} MATCHES "OpenMesh")
    include(OpenMeshPackage)
endif()

include (ACGOutput)

# ========================================================================
# Definitions
# ========================================================================

if (WIN32)
  add_definitions(
      -D_USE_MATH_DEFINES -DNOMINMAX
      -D_CRT_SECURE_NO_WARNINGS
  )
endif ()

set(NO_DECREMENT_DEPRECATED_WARNINGS OFF CACHE BOOL "Disables all deprecated warnings warning about decrement operations on normal circulators.")
if(NO_DECREMENT_DEPRECATED_WARNINGS)
  add_definitions( -DNO_DECREMENT_DEPRECATED_WARNINGS )
endif()

# ========================================================================
# Windows build style control
# ========================================================================

if ( WIN32 )
  if ( NOT DEFINED OPENMESH_BUILD_SHARED )
    set( OPENMESH_BUILD_SHARED false CACHE BOOL "Build as shared library(DLL)?" )
  endif()
endif()

# ========================================================================
# Add bundle targets here
# ========================================================================
if ( NOT DEFINED BUILD_APPS )
  set( BUILD_APPS true CACHE BOOL "Enable or disable building of apps" )
endif()

# Only call fixbundle, when we are building OpenMesh standalone
if( (${PROJECT_NAME} MATCHES "OpenMesh") AND BUILD_APPS )

  if (WIN32)
    if ( NOT "${CMAKE_GENERATOR}" MATCHES "MinGW Makefiles" AND BUILD_APPS )
      add_custom_target (fixbundle ALL
        COMMAND ${CMAKE_COMMAND} -P "${CMAKE_BINARY_DIR}/fixbundle.win.cmake" )
    endif()
  endif()

  if (APPLE)
    add_custom_target (fixbundle ALL
      COMMAND ${CMAKE_COMMAND} -P "${CMAKE_BINARY_DIR}/fixbundle.cmake"
    )
  endif()

endif() # project OpenMesh

# ========================================================================
# Call the subdirectories with there projects
# ========================================================================

add_subdirectory (src/OpenMesh/Core)
add_subdirectory (src/OpenMesh/Tools)
add_subdirectory (src/OpenMesh/Apps)

set(OPENMESH_BENCHMARK_DIR CACHE PATH "Source path of benchmark (https://github.com/google/benchmark).")
if (OPENMESH_BENCHMARK_DIR)
	add_subdirectory(${OPENMESH_BENCHMARK_DIR} benchmark)
	add_subdirectory(src/Benchmark)
endif()

# Do not build unit tests when build as external library
if(${PROJECT_NAME} MATCHES "OpenMesh")
    add_subdirectory (src/Unittests)
else()
	# If built as a dependent project simulate effects of
	# successful finder run:
	set (OPENMESH_FOUND true PARENT_SCOPE)
	set (OPENMESH_LIBRARIES OpenMeshCore OpenMeshTools PARENT_SCOPE)
	set (OPENMESH_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/src" PARENT_SCOPE)

	# Also define variables provided by the old legacy finder.
	set (OPENMESH_CORE_LIBRARY OpenMeshCore PARENT_SCOPE)
	set (OPENMESH_TOOLS_LIBRARY OpenMeshTools PARENT_SCOPE)
	set (OPENMESH_TOOLS_LIBRARY OpenMeshTools PARENT_SCOPE)
	set (OPENMESH_LIBRARY OpenMeshCore OpenMeshTools PARENT_SCOPE)
	get_target_property(_OPENMESH_LIBRARY_DIR OpenMeshCore LIBRARY_OUTPUT_DIRECTORY)
	set (OPENMESH_LIBRARY_DIR "${_OPENMESH_LIBRARY_DIR}" CACHE PATH "The directory where the OpenMesh libraries can be found.")
endif()

add_subdirectory (Doc)

# ========================================================================
# Include Python interface
# ========================================================================

add_subdirectory (src/Python)

# ========================================================================
# Bundle generation (Targets exist, now configure them)
# ========================================================================

# Only call fixbundle, when we are building OpenMesh standalone
if(${PROJECT_NAME} MATCHES "OpenMesh")

  if (WIN32 AND  BUILD_APPS )
    # prepare bundle generation cmake file and add a build target for it
    configure_file ("${CMAKE_SOURCE_DIR}/cmake/fixbundle.cmake.win.in"
       "${CMAKE_BINARY_DIR}/fixbundle.win.cmake" @ONLY IMMEDIATE)

    if ( NOT "${CMAKE_GENERATOR}" MATCHES "MinGW Makefiles" )
      # let bundle generation depend on all targets
      add_dependencies (fixbundle QtViewer DecimaterGui)
    endif()
 
  endif()
 
  # On apple we do a fixbundle, which is only necessary for the apps and not for the libs
  if (APPLE AND BUILD_APPS)
    # prepare bundle generation cmake file and add a build target for it
    configure_file ("${CMAKE_SOURCE_DIR}/cmake/fixbundle.cmake.in"
       "${CMAKE_BINARY_DIR}/fixbundle.cmake" @ONLY IMMEDIATE)

    # let bundle generation depend on all targets
    add_dependencies (fixbundle DecimaterGui ProgViewer QtViewer SubdividerGui) 

    # Required for Snow leopard, and the latest qt. Then the resources have to be copied
    if ( EXISTS "/opt/local/libexec/qt4-mac/lib/QtGui.framework/Versions/4/Resources/qt_menu.nib" )
        add_custom_command(TARGET OpenMesh POST_BUILD
           COMMAND ${CMAKE_COMMAND} -E copy_directory "/opt/local/libexec/qt4-mac/lib/QtGui.framework/Versions/4/Resources/qt_menu.nib" 
                                                      "${CMAKE_BINARY_DIR}/Build/Libraries/qt_menu.nib" )
    endif ()
  endif ()

endif()

# ========================================================================


# display results
acg_print_configure_header (OPENMESH "OpenMesh")
