#-------------------------------------
# MEOS Main CMake file
#-------------------------------------

# Minimum Cmake version supporting fixtures
cmake_minimum_required(VERSION 3.12)

# Disallow in-source builds
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "In-source builds not allowed.
  Please make a new directory (called a build directory) and run CMake from there.
  You may need to remove 'CMakeCache.txt' and 'CMakeFiles/'.")
endif()

# Set a default build type if none was specified
# https://www.kitware.com/cmake-and-the-default-build-type/
set(default_build_type "Release")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Specify search path for CMake modules to be loaded
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
include(CheckSymbolExists)
include(CheckCXXSymbolExists)
include(CheckCSourceRuns)
include(CMakeDependentOption)

#-------------------------------------
# MEOS definitions
#-------------------------------------

# Windows build with MSVC

if(CMAKE_GENERATOR STREQUAL "Visual Studio 17 2022")
  message(STATUS "Setting options for Visual Studio")
  set(MSVC 1)
  # Disable native language support
  add_definitions(-DENABLE_NLS=0)
  # Disable warning C4996: 'strcpy': This function or variable may be unsafe.
  add_definitions(-D_CRT_SECURE_NO_WARNINGS=1)
endif()

# Option for including circular buffers
option(CBUFFER
  "Set ON|OFF (default=OFF) to include circular buffers
  "
  OFF
  )

# Option for including network points
option(NPOINT
  "Set ON|OFF (default=ON) to include network points
  "
  ON
  )

# Option for including geoposes
option(POSE
  "Set ON|OFF (default=OFF) to include geoposes
  "
  OFF
  )

# Option for including rigid geometries, which depend on geoposes
CMAKE_DEPENDENT_OPTION(RGEO
  "Set ON|OFF (default=OFF) to include rigid geometries
  "
  ON "POSE"
  OFF
  )

if(CBUFFER)
  message(STATUS "Including circular buffers")
  add_definitions(-DCBUFFER=1)
else()
  message(STATUS "Excluding circular buffers")
  add_definitions(-DCBUFFER=0)
endif()

if(NPOINT)
  message(STATUS "Including network points")
  add_definitions(-DNPOINT=1)
else()
  message(STATUS "Excluding network points")
  add_definitions(-DNPOINT=0)
endif()

if(POSE)
  message(STATUS "Including geoposes")
  add_definitions(-DPOSE=1)
else()
  message(STATUS "Excluding geoposes")
  add_definitions(-DPOSE=0)
endif()

if(RGEO)
  message(STATUS "Including rigid geometries")
  add_definitions(-DRGEO=1)
else()
  message(STATUS "Excluding rigid geometries")
  add_definitions(-DRGEO=0)
endif()

# Get the MEOS major/minor/micro versions from the text file
file(READ meos_version.txt ver)
string(REGEX MATCH "MEOS_MAJOR_VERSION=([0-9]+)" _ ${ver})
set(MEOS_MAJOR_VERSION ${CMAKE_MATCH_1})
string(REGEX MATCH "MEOS_MINOR_VERSION=([0-9]+)" _ ${ver})
set(MEOS_MINOR_VERSION ${CMAKE_MATCH_1})
string(REGEX MATCH "MEOS_MICRO_VERSION=([0-9]+)" _ ${ver})
set(MEOS_MICRO_VERSION ${CMAKE_MATCH_1})
string(REGEX MATCH "MEOS_MICRO_VERSION=[0-9]+(beta?[0-9]+)" _ ${ver})
set(PROJECT_VERSION_DEV ${CMAKE_MATCH_1})

# Set the project name and project version
project(MEOS VERSION ${MEOS_MAJOR_VERSION}.${MEOS_MINOR_VERSION}.${MEOS_MICRO_VERSION})
message(STATUS "Building MEOS version ${PROJECT_VERSION}")
# Enable testing, must follow project() call above
include(CTest)
# Make CTest run verbose for the output of the failed tests
set(CMAKE_CTEST_ARGUMENTS "--output-on-failure")
enable_testing()

set(MEOS_VERSION "${PROJECT_VERSION}")
set(MEOS_VERSION_STRING "${CMAKE_PROJECT_NAME} ${PROJECT_VERSION}${PROJECT_VERSION_DEV}")
add_definitions(-DMOBILITYDB_VERSION_STRING="${MEOS_VERSION_STRING}")
message(STATUS "MEOS_VERSION_STRING = '${MEOS_VERSION_STRING}'")

# Comment out code used for debugging purposes so it is not concerned by the coverage
if(CMAKE_BUILD_TYPE MATCHES Debug)
  message(STATUS "CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}")
  add_definitions(-DDEBUG_BUILD)
endif()

#-------------------------------------
# Set compiler flags
#-------------------------------------

# Generate position-independent code (PIC)
include(CheckCCompilerFlag)
if(NOT WIN32)
  CHECK_C_COMPILER_FLAG("-fPIC" C_COMPILER_SUPPORTS_FPIC)
  if(C_COMPILER_SUPPORTS_FPIC)
    message(STATUS "Compiling into position-independent code (PIC)")
    add_definitions(-fPIC)
  endif()
endif()

# FFSL functions for finding the first (least significant) bit
check_symbol_exists(ffsl "string.h" HAS_FFSL)
if(NOT HAS_FFSL)
  message(STATUS "No ffsl functions provided")
  add_definitions(-DNO_FFSL)
endif()

# Added to build on Windows to avoid the error 'M_PI': undeclared identifier
add_compile_definitions(_USE_MATH_DEFINES)

# Set default compile flags and code coverage for GCC
if(CMAKE_COMPILER_IS_GNUCC)
  # MEOS compiler flags
  add_definitions(-Wall -Wextra -std=gnu1x -Wunused-parameter -Wno-strict-aliasing)
  # Code coverage
  if(WITH_COVERAGE)
    message(STATUS "Generating code coverage")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -fprofile-arcs -ftest-coverage")
  endif()
endif()

# Set compiler optimizations for release
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O2")
# Disable compiler optimizations for debugging
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O0")

# Check machine architecture: big endian vs. little endian
# Needed for WKB support in MEOS and PostGIS (file postgis_config.h.in)
include(TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
  message(STATUS "BIG_ENDIAN")
  add_definitions(-DMEOS_IS_BIG_ENDIAN=1)
  set(DEF_WORDS_BIGENDIAN ON)
else()
  message(STATUS "LITTLE_ENDIAN")
  add_definitions(-DMEOS_IS_BIG_ENDIAN=0)
  unset(DEF_WORDS_BIGENDIAN)
endif()

# check for needed Header files and functions
set(CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
check_symbol_exists(strchrnul "string.h" HAVE_STRCHRNUL)
unset(CMAKE_REQUIRED_DEFINITIONS)
check_symbol_exists(__get_cpuid "cpuid.h" HAVE__GET_CPUID)
check_c_source_runs("
#include <stdint.h>
int main() {
    uint64_t x = 1, r;
    __asm__ (\"popcntq %1, %0\" : \"=r\"(r) : \"r\"(x));
    return 0;
}" HAVE_X86_64_POPCNTQ)

#--------------------------------
# Other dependencies
#--------------------------------

# GEOS geometry library
find_package(GEOS REQUIRED)
include_directories(SYSTEM ${GEOS_INCLUDE_DIR})
math(EXPR POSTGIS_GEOS_VERSION "${GEOS_VERSION_MAJOR} * 10000 + ${GEOS_VERSION_MINOR} * 100 + ${GEOS_VERSION_MICRO}")
message(STATUS "POSTGIS_GEOS_VERSION: ${POSTGIS_GEOS_VERSION}")

# Proj reprojection library
find_package(PROJ REQUIRED)
include_directories(SYSTEM ${PROJ_INCLUDE_DIRS})
math(EXPR POSTGIS_PROJ_VERSION "${PROJ_VERSION_MAJOR} * 10 + ${PROJ_VERSION_MINOR}")
message(STATUS "POSTGIS_PROJ_VERSION: ${POSTGIS_PROJ_VERSION}")

# JSON-C library (used for MF-JSON input/output)
find_package(JSON-C REQUIRED)
include_directories(SYSTEM ${JSON-C_INCLUDE_DIRS})
message(STATUS "JSON-C_INCLUDE_DIRS: ${JSON-C_INCLUDE_DIRS}")
message(STATUS "JSON-C_LIBRARIES: ${JSON-C_LIBRARIES}")

# GSL (GNU Scientific Library) (used for skiplists)
find_package(GSL REQUIRED)
include_directories(SYSTEM ${GSL_INCLUDE_DIRS})
message(STATUS "GSL_VERSION: ${GSL_VERSION}")
add_definitions(-DGSL_VERSION_STRING="${GSL_VERSION}")

#----------------------------------------------
# Configure PostGIS with optional dependencies
#----------------------------------------------

find_package(GDAL QUIET)
if(GDAL_FOUND)
  message(STATUS "GDAL version '${GDAL_VERSION}' found")
else()
  message(STATUS "GDAL not found")
endif()

find_package(LibXml2)
if(LibXml2_FOUND)
  message(STATUS "LibXml2 version ${LIBXML2_VERSION_STRING} found")
else()
  message(STATUS "LibXml2 not found")
endif()

# find_package(Protobuf REQUIRED)
find_package(Protobuf QUIET)
if(Protobuf_FOUND)
  # Note that ${Protobuf_VERSION_STRING} is not set
  if (Protobuf_INCLUDE_DIR AND EXISTS "${Protobuf_INCLUDE_DIR}/google/protobuf/stubs/common.h")
      set(_Protobuf_VERSION_REGEX "^#define[ \t]+GOOGLE_PROTOBUF_VERSION[ \t]+([0-9]+).*$")
      file(STRINGS "${Protobuf_INCLUDE_DIR}/google/protobuf/stubs/common.h" Protobuf_VERSION REGEX "${_Protobuf_VERSION_REGEX}")
      if(Protobuf_VERSION MATCHES ${_Protobuf_VERSION_REGEX})
          set(Protobuf_VERSION "${CMAKE_MATCH_1}")
          math(EXPR Protobuf_VERSION_MAJOR "${Protobuf_VERSION} / 1000000")
          math(EXPR Protobuf_VERSION_MINOR "${Protobuf_VERSION} / 1000 % 1000")
          math(EXPR Protobuf_VERSION_PATCH "${Protobuf_VERSION} % 1000")
          set(Protobuf_VERSION_STRING "${Protobuf_VERSION_MAJOR}.${Protobuf_VERSION_MINOR}.${Protobuf_VERSION_PATCH}")
      endif()
      message(STATUS "Protobuf version '${Protobuf_VERSION_STRING}' found")
      unset(_Protobuf_VERSION_REGEX)
  endif()
else()
  message(STATUS "Protobuf not found")
endif()

find_package(Gettext QUIET)
if(GETTEXT_FOUND)
  message(STATUS "GETTEXT version '${GETTEXT_VERSION_STRING}' found")
else()
  message(STATUS "GETTEXT not found")
endif()

find_package(ICU QUIET)
if(ICU_FOUND)
  message(STATUS "ICU Version '${ICU_VERSION}' found")
else()
  message(STATUS "ICU NOT found")
endif()

#-------------------------------------
# Build the MEOS library
#-------------------------------------

message(STATUS "-------------------------")
message(STATUS "Building the MEOS library")
message(STATUS "-------------------------")

# Set MEOS option for building the MEOS library
set(MEOS ON)

# Enable code specific to MEOS, typically user-oriented wrapper functions
# hiding internal implementation details from the API. Such functions are
# not used in the MobilityDB PostgreSQL extension.
add_definitions(-DMEOS=1)

# Set the version and name of the MEOS library
set(MEOS_LIB_VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}")
set(MEOS_LIB_NAME "meos")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

# Set the location of the directory of the time zone database
add_definitions(-DSYSTEMTZDIR="/usr/share/zoneinfo")
message(STATUS "Directory of the time zone database: /usr/share/zoneinfo")

# Option to build MEOS as a shared or static library
option(BUILD_SHARED_LIBS
  "Set BUILD_SHARED_LIBS (default=ON) to build MEOS as a shared library"
  ON
)

# Option to show debug messages for analyzing the expandable data structures
option(DEBUG_EXPAND
  "Set DEBUG_EXPAND (default=OFF) to show debug messages for analyzing the
  expandable data structures
  "
  OFF
)

if(DEBUG_EXPAND)
  add_definitions(-DDEBUG_EXPAND=1)
  message(STATUS "Showing debug messages for expandable data structures")
endif()

#-----------------------------------------------------------------------------
# Install directories
#-----------------------------------------------------------------------------

include(GNUInstallDirs)

#-----------------------------------------
# Set default PostgreSQL Version for MEOS
#-----------------------------------------

set(POSTGRESQL_VERSION_STRING "PostgreSQL 14.2")
set(POSTGRESQL_VERSION_NUMBER "140200")
set(POSTGRESQL_VERSION_MAJOR "14")
set(POSTGRESQL_VERSION_MINOR "2")

# MobilityDB definitions for PostgreSQL version-dependent code
add_definitions(-DPOSTGRESQL_VERSION_STRING="${POSTGRESQL_VERSION_STRING}")
add_definitions(-DPOSTGRESQL_VERSION_NUMBER=${POSTGRESQL_VERSION_NUMBER})

#--------------------------------------
# Set default PostGIS Version for MEOS
#--------------------------------------

set(POSTGIS_VERSION_STRING "PostGIS 3.5.0")
set(POSTGIS_VERSION_NUMBER "30500")
set(POSTGIS_VERSION_MAJOR "3")
set(POSTGIS_VERSION_MINOR "5")
set(POSTGIS_VERSION_MICRO "0")
set(POSTGIS_VERSION "3.5.0")

set(SRID_MAX "999999")
set(SRID_USR_MAX "998999")

# POSTGIS_PGSQL_VERSION is used in PostGIS for PostgreSQL version-dependent code
# Note that we cannot add ${POSTGRESQL_VERSION_MINOR} since, e.g., for version
# 13.13 it will yield 143 which is not intended
math(EXPR POSTGIS_PGSQL_VERSION "${POSTGRESQL_VERSION_MAJOR} * 10")
message(STATUS "POSTGIS_PGSQL_VERSION=${POSTGIS_PGSQL_VERSION}")

add_definitions(-DPOSTGIS_VERSION_STRING="${POSTGIS_VERSION_STRING}")
add_definitions(-DPOSTGIS_VERSION_NUMBER=${POSTGIS_VERSION_NUMBER})
add_definitions(-DPOSTGIS_PGSQL_VERSION=${POSTGIS_PGSQL_VERSION})

#--------------------------------
# MEOS directories
#--------------------------------

# PostgreSQL - detect host-specific values and render pg_config.h.
# See cmake/ConfigurePgConfig.cmake for the rationale: the previous
# postgres/pg_config.h.in was a Linux snapshot checked in as a static
# file; this module does what PG's own ./configure would do, narrowed to
# the macros MEOS/MobilityDB actually references.
include(${CMAKE_SOURCE_DIR}/cmake/ConfigurePgConfig.cmake)
configure_file(
  ${CMAKE_SOURCE_DIR}/postgres/pg_config.h.in
  ${CMAKE_BINARY_DIR}/postgres/pg_config.h
  @ONLY)
include_directories(SYSTEM "${CMAKE_SOURCE_DIR}/postgres/timezone")
include_directories(SYSTEM "${CMAKE_SOURCE_DIR}/postgres")
include_directories(SYSTEM "${CMAKE_BINARY_DIR}/postgres")
add_subdirectory("${CMAKE_SOURCE_DIR}/postgres" "postgres")

# PostGIS
configure_file(${CMAKE_SOURCE_DIR}/postgis/liblwgeom/liblwgeom.h.in ${CMAKE_BINARY_DIR}/postgis/liblwgeom/liblwgeom.h)
configure_file(${CMAKE_SOURCE_DIR}/postgis/postgis_config.h.in ${CMAKE_BINARY_DIR}/postgis/postgis_config.h)
include_directories(SYSTEM "${CMAKE_SOURCE_DIR}/postgis/liblwgeom")
include_directories(SYSTEM "${CMAKE_SOURCE_DIR}/postgis")
include_directories(SYSTEM "${CMAKE_BINARY_DIR}/postgis/liblwgeom")
include_directories(SYSTEM "${CMAKE_BINARY_DIR}/postgis")
add_subdirectory("${CMAKE_SOURCE_DIR}/postgis" "postgis")

# MEOS
include_directories("${CMAKE_SOURCE_DIR}/include")
add_subdirectory("src")

# Must be after add_library
if(CMAKE_GENERATOR STREQUAL "Unix Makefiles" OR CMAKE_GENERATOR STREQUAL "Ninja")
  # Disable all warnings on embeded code from PostgreSQL and PostGIS
  target_compile_options(liblwgeom PRIVATE "-w")
  target_compile_options(ryu PRIVATE "-w")
  target_compile_options(timezone PRIVATE "-w")
  target_compile_options(utils PRIVATE "-w")
elseif(CMAKE_GENERATOR STREQUAL "Visual Studio 17 2022")
  # Disable all warnings on embedded code from PostgreSQL and PostGIS
  target_compile_options(liblwgeom PRIVATE "/W0")
  target_compile_options(ryu PRIVATE "/W0")
  target_compile_options(timezone PRIVATE "/W0")
  target_compile_options(utils PRIVATE "/W0")
  # Enable all warnings on MEOS
  target_compile_options(temporal PRIVATE "/W4")
  target_compile_options(geo PRIVATE "/W4")
  if(CBUFFER)
    target_compile_options(cbuffer PRIVATE "/W4")
  endif()
  if(NPOINT)
    target_compile_options(npoint PRIVATE "/W4")
  endif()
  if(POSE)
    target_compile_options(pose PRIVATE "/W4")
  endif()
  if(RGEO)
    target_compile_options(rgeo PRIVATE "/W4")
  endif()
endif()

#--------------------------------
# Build MEOS library
#--------------------------------

# Customize the meos.in.h and meos_internal.in.h files for external include with MEOS library
# Read files
file(READ "${CMAKE_SOURCE_DIR}/include/meos.h" MEOS_H)
file(READ "${CMAKE_SOURCE_DIR}/include/meos_internal.h" MEOS_INTERNAL_H)
file(READ "${CMAKE_SOURCE_DIR}/include/meos_geo.h" MEOS_GEO_H)
file(READ "${CMAKE_SOURCE_DIR}/include/postgres_ext_defs.in.h" POSTGRES_EXT_DEFS_IN_H)
file(READ "${CMAKE_SOURCE_DIR}/include/postgis_ext_defs.in.h" POSTGIS_EXT_DEFS_IN_H)
# Generate meos_export.h
string(REGEX REPLACE "#if MEOS\n#include \"postgres_int_defs.h\"\n#else\n#include <postgres.h>\n#include <utils/date.h>\n#include <utils/timestamp.h>\n#endif" "${POSTGRES_EXT_DEFS_IN_H}" MEOS_EXP_H "${MEOS_H}")
file(WRITE "${CMAKE_BINARY_DIR}/meos_export.h" "${MEOS_EXP_H}")
# Generate meos_internal_export.h
string(REGEX REPLACE "#include \"temporal/meos_catalog.h\"" "#include \"meos_catalog.h\"" MEOS_INTERNAL_EXP_H "${MEOS_INTERNAL_H}")
file(WRITE "${CMAKE_BINARY_DIR}/meos_internal_export.h" "${MEOS_INTERNAL_EXP_H}")
# Generate meos_geo_export.h
string(REGEX REPLACE "#include <liblwgeom.h>" "${POSTGIS_EXT_DEFS_IN_H}" MEOS_GEO_EXP_H "${MEOS_GEO_H}")
file(WRITE "${CMAKE_BINARY_DIR}/meos_geo_export.h" "${MEOS_GEO_EXP_H}")

# MEOS
set(PROJECT_OBJECTS "$<TARGET_OBJECTS:temporal>")
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:geo>")
if(CBUFFER)
  message(STATUS "Including circular buffers")
  set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:cbuffer>")
endif()
if(NPOINT)
  message(STATUS "Including network points")
  set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:npoint>")
endif()
if(POSE)
  message(STATUS "Including geoposes")
  set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:pose>")
endif()
if(RGEO)
  message(STATUS "Including rigid geometries")
  set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:rgeo>")
endif()

# PostgreSQL
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:common>")
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:port>")
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:timezone>")
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:utils>")

# Build the library: PostGIS
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:liblwgeom>")
set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:ryu>")

# Build the library: All
add_library(${MEOS_LIB_NAME} ${PROJECT_OBJECTS})
if(APPLE)
  set_target_properties(${MEOS_LIB_NAME} PROPERTIES
    LINK_FLAGS "-Wl,-undefined,dynamic_lookup")
endif()

#--------------------------------
# Specify libraries to link
#--------------------------------

target_link_libraries(${MEOS_LIB_NAME} ${JSON-C_LIBRARIES})
target_link_libraries(${MEOS_LIB_NAME} ${GEOS_LIBRARY})
target_link_libraries(${MEOS_LIB_NAME} ${PROJ_LIBRARIES})
target_link_libraries(${MEOS_LIB_NAME} ${GSL_LIBRARY})
target_link_libraries(${MEOS_LIB_NAME} ${GSL_CBLAS_LIBRARY})

#--------------------------------
# Install rules
#--------------------------------

install(FILES "${CMAKE_BINARY_DIR}/meos_export.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  RENAME "meos.h")
install(
  FILES "${CMAKE_BINARY_DIR}/meos_internal_export.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  RENAME "meos_internal.h")
install(
  FILES "${CMAKE_BINARY_DIR}/meos_geo_export.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  RENAME "meos_geo.h")
install(
  FILES "${CMAKE_SOURCE_DIR}/include/meos_internal_geo.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
install(
  FILES "${CMAKE_SOURCE_DIR}/src/geo/spatial_ref_sys.csv"
  DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}")

if(CBUFFER)
  install(
    FILES "${CMAKE_SOURCE_DIR}/include/meos_cbuffer.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
endif()
if(NPOINT)
  install(
    FILES "${CMAKE_SOURCE_DIR}/include/meos_npoint.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  install(
    FILES "${CMAKE_SOURCE_DIR}/examples/data/ways.csv"
    DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}")
  install(
    FILES "${CMAKE_SOURCE_DIR}/examples/data/ways1000.csv"
    DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}")
endif()
if(POSE)
  install(
    FILES "${CMAKE_SOURCE_DIR}/include/meos_pose.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
endif()
if(RGEO)
  install(
    FILES "${CMAKE_SOURCE_DIR}/include/meos_rgeo.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
endif()

install(
  FILES "${CMAKE_SOURCE_DIR}/include/temporal/meos_catalog.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
install(TARGETS ${MEOS_LIB_NAME} DESTINATION "${CMAKE_INSTALL_LIBDIR}")
message(STATUS "Building MEOS:")
message(STATUS "  Install prefix: '${CMAKE_INSTALL_PREFIX}'")
message(STATUS "  Library file: '${CMAKE_INSTALL_LIBDIR}'")
message(STATUS "  Include file: '${CMAKE_INSTALL_INCLUDEDIR}'")

#-----------------------------------------------------------------------------
# Configure pkg-config file meos.pc
#-----------------------------------------------------------------------------

# Consider CMAKE_INSTALL_PREFIX with spaces
string(REPLACE " " "\\ " prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "$\{prefix\}")
if(IS_ABSOLUTE "${CMAKE_INSTALL_INCLUDEDIR}")
  set(includedir "${CMAKE_INSTALL_INCLUDEDIR}")
else()
  set(includedir "$\{prefix\}/${CMAKE_INSTALL_INCLUDEDIR}")
endif()
if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
  set(libdir "${CMAKE_INSTALL_LIBDIR}")
else()
  set(libdir "$\{exec_prefix\}/${CMAKE_INSTALL_LIBDIR}")
endif()
set(VERSION ${MEOS_LIB_VERSION})

configure_file(
  ${CMAKE_SOURCE_DIR}/tools/meos.pc.in
  ${CMAKE_BINARY_DIR}/meos.pc
  @ONLY)

install(
  FILES ${CMAKE_BINARY_DIR}/meos.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

#-----------------------------------------------------------------------------
# The End
#-----------------------------------------------------------------------------
