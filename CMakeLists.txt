cmake_minimum_required(VERSION 3.10)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

project(sally CXX C)
enable_testing()

# Set CMake policies to suppress warnings
cmake_policy(SET CMP0167 NEW)
cmake_policy(SET CMP0135 NEW)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror -Wall")

# If ENABLE_COVERAGE is defined, try to set coverage flags.
if (ENABLE_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
endif()

# Default is release with debug info
if(NOT CMAKE_BUILD_TYPE)
set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

# Add the target for the check
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})

# Find the Boost libraries
find_package(Boost 1.53.0 COMPONENTS program_options iostreams thread system REQUIRED)
include_directories(${Boost_INCLUDE_DIRS})

# Some settings based on word size
if (CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(ANTLR_CONFIG_FLAG "--enable-64bit")
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(ANTLR_CONFIG_FLAG "")
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)

# Find the GMP number library
find_path(GMP_INCLUDE NAMES gmp.h)
if (SALLY_STATIC_BUILD)
  find_library(GMP_LIBRARY libgmp.a gmp)
else()
  find_library(GMP_LIBRARY gmp)
endif()
if (GMP_INCLUDE AND GMP_LIBRARY)
  message(STATUS "GMP library: " ${GMP_LIBRARY})
else()
  message(FATAL_ERROR "Could not find the GMP number library (sudo apt-get install libgmp-dev)")
endif()

# Find CUDD
SET(CUDD_HOME CACHE STRING "Cudd installation directory")
find_package(CUDD 3.0.0)
if (NOT CUDD_FOUND)
  message(FATAL_ERROR "Could not find the Cudd library")
endif()

# Find LibPoly
SET(LIBPOLY_HOME CACHE STRING "LibPoly installation directory")
find_package(LibPoly 0.1.5)
if (LIBPOLY_FOUND)
  add_definitions(-DWITH_LIBPOLY)
  include_directories(${LIBPOLY_INCLUDE_DIR})
endif()

# Find Yices if LibPoly is there
if (LIBPOLY_FOUND)
  SET(YICES2_HOME CACHE STRING "Yices2 installation directory")
  find_package(Yices2 2.6.0)
  if (YICES2_FOUND)
    add_definitions(-DWITH_YICES2)
    include_directories(${YICES2_INCLUDE_DIR})
  endif()
endif()

# Find MathSAT5
SET(MATHSAT5_HOME CACHE STRING "MathSAT5 installation directory")
find_package(MathSAT5 5.3.3)
if (MATHSAT5_FOUND)
  add_definitions(-DWITH_MATHSAT5)
  include_directories(${MATHSAT5_INCLUDE_DIR})
endif()

# Find DREAL
SET(DREAL_HOME CACHE STRING "DReal installation directory")
find_package(Dreal)
if (DREAL_FOUND)
  add_definitions(-DWITH_DREAL)
  #add_definitions(${DREAL_DEFINITIONS})
  include_directories(${DREAL_INCLUDE_DIRS})
  #message(STATUS "Dreal definitions: ${DREAL_DEFINITIONS}")
endif()

# Find Z3
SET(Z3_HOME CACHE STRING "Z3 installation directory")
find_package(Z3 4.8)
if (Z3_FOUND)
  add_definitions(-DWITH_Z3)
  include_directories(${Z3_INCLUDE_DIR})
endif()

# Find OpenSMT2
SET(OPENSMT2_HOME CACHE STRING "OpenSMT2 installation directory")
find_package(OpenSMT2)
if (OPENSMT2_FOUND)
  add_definitions(-DWITH_OPENSMT2)
  include_directories(${OPENSMT2_INCLUDE_DIR})
endif()

# Find Btor2Tools
SET(BTOR2TOOLS_HOME CACHE STRING "Btor2Tools installation directory")
find_package(Btor2Tools)
if (BTOR2TOOLS_FOUND)
  add_definitions(-DWITH_BTOR2TOOLS)
  include_directories(${BTOR2TOOLS_INCLUDE_DIRS})
  set(CMAKE_INSTALL_RPATH "${BTOR2TOOLS_LIBRARY_DIR}")
  set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
  set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
  set(CMAKE_INSTALL_RPATH "${BTOR2TOOLS_HOME}/build/lib")
endif()

# Make sure antlr C runtime is here
include(ExternalProject)
ExternalProject_Add(
   libantlr3c-3.4
   URL "${sally_SOURCE_DIR}/antlr/libantlr3c-3.4.tar.gz"
   URL_MD5 08b1420129d5dccd0f4461cedf2a0d7d
   DOWNLOAD_EXTRACT_TIMESTAMP TRUE
   PATCH_COMMAND chmod -R u+w src include
         COMMAND patch -p1 -t -N < "${sally_SOURCE_DIR}/antlr/libantlr3c-3.4.patch"
   CONFIGURE_COMMAND <SOURCE_DIR>/configure ${ANTLR_CONFIG_FLAG} --enable-debuginfo --disable-antlrdebug --prefix=<INSTALL_DIR>
   BUILD_IN_SOURCE 1
)
ExternalProject_Get_Property(libantlr3c-3.4 INSTALL_DIR)
set(ANTLR3C_INCLUDE_DIR "${INSTALL_DIR}/include")
set(ANTLR3C_LIBRARY_DIR "${INSTALL_DIR}/lib")
set(ANTLR3C_LIBRARY "antlr3c")
add_library(libantlr3c STATIC IMPORTED)
set_property(TARGET libantlr3c PROPERTY IMPORTED_LOCATION ${ANTLR3C_LIBRARY_DIR}/libantlr3c.a)

# The antl3 binary
set(ANTLR "${sally_SOURCE_DIR}/antlr/antlr3")

# Add includes
include_directories(${sally_SOURCE_DIR}/src ${ANTLR3C_INCLUDE_DIR})

# Configure the subdirectories
add_subdirectory(src)

# Add the test project
add_subdirectory(test)
