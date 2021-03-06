# Minimum version required
cmake_minimum_required (VERSION 3.2)

# Project name
project (qdldl)

add_compile_options(-Wall -Wextra)

# Set the output folder where your program will be created
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/out)
set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/out)

# Some non-standard CMake modules
LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/configure/cmake)

# Export compilation commands for IDEs and autocompletion
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Options
# ----------------------------------------------
# Use floats instead of doubles
set(DFLOAT OFF CACHE BOOL "Use float numbers instead of doubles")
message(STATUS "Floats are ${DFLOAT}")

# Use long integers for indexing
set(DLONG ON CACHE BOOL "Use long integers (64bit) for indexing")
if (NOT (CMAKE_SIZEOF_VOID_P EQUAL 8))
	message(STATUS "Disabling long integers (64bit) on 32bit machine")
	set(DLONG OFF)
endif()
message(STATUS "Long integers (64bit) are ${DLONG}")


# Set Compiler flags
# ----------------------------------------------
set(CMAKE_POSITION_INDEPENDENT_CODE ON)  # -fPIC


# Add compiler options if we are not on windows
if (NOT MSVC)

    if (COVERAGE)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage")
	if(FORTRAN)
		set(CMAKE_FORTRAN_FLAGS "${CMAKE_FORTRAN_FLAGS} --coverage")
	endif(FORTRAN)
    endif()

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O0 -g")
endif (NOT MSVC)

# Generate header file with the global options
# ---------------------------------------------

# numeric types
if(DFLOAT)
  set(QDLDL_FLOAT_TYPE "float")
else()
	set(QDLDL_FLOAT_TYPE "double")
endif()

if(DLONG)
  set(QDLDL_INT_TYPE "long long")
else()
	set(QDLDL_INT_TYPE "int")
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/configure/qdldl_types.h.in
               ${CMAKE_CURRENT_SOURCE_DIR}/include/qdldl/qdldl_types.h
               NEWLINE_STYLE LF)

option(BUILD_CATKIN_PACKAGE "Build catkin package" ON)

if(BUILD_CATKIN_PACKAGE)

  # Find Catkin
  find_package(catkin REQUIRED COMPONENTS)

  # Declare this project as a catkin package
  catkin_package(
    INCLUDE_DIRS include
    LIBRARIES ${PROJECT_NAME} qdldlobject
    CATKIN_DEPENDS
  )

  set(CMAKE_INSTALL_BINDIR "${CATKIN_PACKAGE_BIN_DESTINATION}")
  set(CMAKE_INSTALL_LIBDIR "${CATKIN_PACKAGE_LIB_DESTINATION}")
  set(CMAKE_INSTALL_INCLUDEDIR "${CATKIN_PACKAGE_INCLUDE_DESTINATION}")
else()
  set(CMAKE_INSTALL_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/qdldl")
endif(BUILD_CATKIN_PACKAGE)

# Set sources
# ----------------------------------------------
set(
	qdldl_src
	src/qdldl.c
	)

set(
	qdldl_headers
	include/qdldl/qdldl.h
	include/qdldl/qdldl_types.h
	)

# Create object library
# ----------------------------------------------
add_library (qdldlobject ${qdldl_src})
# Declare include directories for the cmake exported target
target_include_directories(qdldlobject PRIVATE ${PROJECT_SOURCE_DIR}/include)

include(GNUInstallDirs)

if(BUILD_CATKIN_PACKAGE)
	# Install qdldl object library
	install(TARGETS qdldlobject
		EXPORT  ${PROJECT_NAME}
        LIBRARY       DESTINATION "${CATKIN_PACKAGE_LIB_DESTINATION}"
        ARCHIVE       DESTINATION "${CATKIN_PACKAGE_LIB_DESTINATION}"
        RUNTIME       DESTINATION "${CATKIN_PACKAGE_BIN_DESTINATION}")
else()
	install(TARGETS qdldlobject
		EXPORT  ${PROJECT_NAME}
		LIBRARY       DESTINATION "${CMAKE_INSTALL_LIBDIR}"
		ARCHIVE       DESTINATION "${CMAKE_INSTALL_LIBDIR}"
		RUNTIME       DESTINATION "${CMAKE_INSTALL_BINDIR}")
endif()

# Create Static Library
# ----------------------------------------------

# Static library
add_library (qdldlstatic STATIC ${qdldl_src} ${qdldl_headers})
# Give same name to static library output
set_target_properties(qdldlstatic PROPERTIES OUTPUT_NAME qdldl)

# Declare include directories for the cmake exported target
target_include_directories(qdldlstatic
                           PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
                                  "$<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}>")

# Install Static Library
# ----------------------------------------------

if(BUILD_CATKIN_PACKAGE)
    install(TARGETS qdldlstatic
        EXPORT  ${PROJECT_NAME}
        LIBRARY       DESTINATION "${CATKIN_PACKAGE_LIB_DESTINATION}"
        ARCHIVE       DESTINATION "${CATKIN_PACKAGE_LIB_DESTINATION}"
        RUNTIME       DESTINATION "${CATKIN_PACKAGE_BIN_DESTINATION}")
else()
  install(TARGETS qdldlstatic
        EXPORT  ${PROJECT_NAME}
        ARCHIVE       DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        LIBRARY       DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        RUNTIME       DESTINATION "${CMAKE_INSTALL_BINDIR}")
endif()

# Install Headers
# ----------------------------------------------
install(FILES ${qdldl_headers} DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")


# Install Shared Library
# ----------------------------------------------
# Create qdldl shared library
add_library (qdldl SHARED ${qdldl_src} ${qdldl_headers})

# Declare include directories for the cmake exported target
target_include_directories(qdldl
	PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
	"$<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}>")

if(BUILD_CATKIN_PACKAGE)
	# Install qdldl object library in catkin workspace
	install(TARGETS qdldl
		EXPORT  ${PROJECT_NAME}
		LIBRARY       DESTINATION "${CATKIN_PACKAGE_LIB_DESTINATION}"
		ARCHIVE       DESTINATION "${CATKIN_PACKAGE_LIB_DESTINATION}"
		RUNTIME       DESTINATION "${CATKIN_PACKAGE_BIN_DESTINATION}")
else()
	# Install qdldl shared library
	install(TARGETS qdldl
		EXPORT  ${PROJECT_NAME}
		LIBRARY       DESTINATION "${CMAKE_INSTALL_LIBDIR}"
		ARCHIVE       DESTINATION "${CMAKE_INSTALL_LIBDIR}"
		RUNTIME       DESTINATION "${CMAKE_INSTALL_BINDIR}")
endif(BUILD_CATKIN_PACKAGE)


# Create demo executable (linked to static library)
add_executable (qdldl_example ${PROJECT_SOURCE_DIR}/examples/c/example.c)
target_link_libraries (qdldl_example qdldlstatic)


# Create CMake packages for the build directory
# ----------------------------------------------
include(CMakePackageConfigHelpers)

export(EXPORT ${PROJECT_NAME}
  FILE "${CMAKE_CURRENT_BINARY_DIR}/qdldl-targets.cmake"
  NAMESPACE qdldl::)

if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/qdldl-config.cmake)
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/qdldl-config.cmake "include(\"\${CMAKE_CURRENT_LIST_DIR}/qdldl-targets.cmake\")\n")
endif()


# Create CMake packages for the install directory
# ----------------------------------------------

set(ConfigPackageLocation lib/cmake/qdldl)

install(EXPORT ${PROJECT_NAME}
        FILE qdldl-targets.cmake
        NAMESPACE qdldl::
        DESTINATION ${ConfigPackageLocation})

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/qdldl-config.cmake
        DESTINATION ${ConfigPackageLocation})



# Add uninstall command
# ----------------------------------------------
if(NOT TARGET uninstall)
    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/configure/cmake/cmake_uninstall.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
        IMMEDIATE @ONLY)

    add_custom_target(uninstall
        COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()



# Add testing
# ----------------------------------------------
# Add custom command to generate tests
if (NOT BUILD_CATKIN_PACKAGE AND UNITTESTS)

    # Add test_headers and codegen_test_headers
    add_subdirectory(tests)

    # Direct qdldl solver testing
    add_executable(qdldl_tester
                ${PROJECT_SOURCE_DIR}/tests/qdldl_tester.c ${PROJECT_SOURCE_DIR}/tests/minunit.h
                ${test_headers})
    target_link_libraries (qdldl_tester qdldlstatic)

    # Add testing
    include(CTest)
    enable_testing()
    add_test(NAME tester COMMAND $<TARGET_FILE:qdldl_tester>)
endif()

if (BUILD_CATKIN_PACKAGE AND CATKIN_ENABLE_TESTING)
  # GTest need cpp
  set_property(SOURCE ${PROJECT_SOURCE_DIR}/tests/qdldl_tester.c PROPERTY LANGUAGE CXX)
  catkin_add_gtest(test_${PROJECT_NAME}
    ${PROJECT_SOURCE_DIR}/tests/qdldl_tester.c
  )
  if(TARGET test_${PROJECT_NAME})
    target_compile_options(test_${PROJECT_NAME} PRIVATE -DUSE_GTEST=ON --std=c++11)
    target_include_directories(test_${PROJECT_NAME} PRIVATE
      include
      ${catkin_INCLUDE_DIRS}
      ${PROJECT_SOURCE_DIR}/tests
    )
    target_link_libraries( test_${PROJECT_NAME}
      qdldlstatic
      ${catkin_LIBRARIES}
      gtest_main
    )
    find_package(cmake_code_coverage QUIET)
    if(cmake_code_coverage_FOUND)
      add_gtest_coverage()
    endif(cmake_code_coverage_FOUND)
  endif()
endif()
