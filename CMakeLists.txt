# Copyright (c) 2011 Cloudera, Inc. All rights reserved.

cmake_minimum_required(VERSION 3.2.3)

# where to find cmake modules
set(CMAKE_MODULE_PATH cmake_modules $ENV{IMPALA_HOME}/cmake_modules)

# compiler flags that are common across debug/release builds
#  -Wall: Enable all warnings.
#  -Wno-sign-compare: suppress warnings for comparison between signed and unsigned
#    integers
#   -fno-strict-aliasing: disable optimizations that assume strict aliasing. This
#       is unsafe to do if the code uses casts (which we obviously do).
#  -Wno-unknown-pragmas: suppress warnings for unknown (compiler specific) pragmas
#  -fsigned-char: on aarch64 platform, type of char default is unsigned char, here
#        set it to signed-char to be compatible with x86-64
#  -Wno-deprecated: gutil contains deprecated headers
#  -Wno-vla: we use C99-style variable-length arrays
#  -pthread: enable multithreaded malloc
#  -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG: enable nanosecond precision for boost
#  -fno-omit-frame-pointers: Keep frame pointer for functions in register
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -march=armv8-a+crc")
endif()
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -Wall -Wno-sign-compare -Wno-unknown-pragmas -pthread")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fno-strict-aliasing -fno-omit-frame-pointer")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fsigned-char")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -std=c++14")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -Wno-deprecated -Wno-vla")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_SYSTEM_NO_DEPRECATED")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -B $ENV{IMPALA_TOOLCHAIN_PACKAGES_HOME}/binutils-$ENV{IMPALA_BINUTILS_VERSION}/bin/")
IF($ENV{USE_GOLD_LINKER} STREQUAL "true")
  SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fuse-ld=gold")
ENDIF()

# For any gcc builds:
#   -g: Enable symbols for profiler tools
#   -Wno-unused-local-typedefs: Do not warn for local typedefs that are unused.
SET(CXX_GCC_FLAGS "-g -Wno-unused-local-typedefs")
if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 7.0)
  # We need to add additional arguments for GCC 7+. We go down this branch if building
  # with a non-GCC compiler of version 7+, but in that case CXX_GCC_FLAGS is not used,
  # so it is inconsequential. TODO: IMPALA-5490: make this non-conditional when we
  # upgrade GCC.
  #  -faligned-new: new will automatically align types. Otherwise "new Counter()" in the
  #       Kudu util code produces a warning (see KUDU-2094).
  #   TODO: -faligned-new is part of C++17, remove flag when we bump language version.
  SET(CXX_GCC_FLAGS "${CXX_GCC_FLAGS} -faligned-new")
endif()

# compiler flags for different build types (run 'cmake -DCMAKE_BUILD_TYPE=<type> .')
# For CMAKE_BUILD_TYPE=Debug
#   -ggdb: Enable gdb debugging
# For CMAKE_BUILD_TYPE=Release
#   -O3: Enable all compiler optimizations
#   -DNDEBUG: Turn off dchecks/asserts/debug only code.
#
# Debug information is stored as dwarf2 to be as compatible as possible
SET(CXX_FLAGS_DEBUG "${CXX_GCC_FLAGS} -ggdb -gdwarf-2")
# -Werror: compile warnings should be errors when using the toolchain compiler.
#   Enabled for DEBUG, ASAN, TSAN and UBSAN builds which are built pre-commit.
SET(CXX_FLAGS_DEBUG "${CXX_FLAGS_DEBUG} -Werror")
SET(CXX_FLAGS_RELEASE "${CXX_GCC_FLAGS} -O3 -DNDEBUG -gdwarf-2")

# if no build build type is specified, default to debug builds
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE DEBUG)
endif(NOT CMAKE_BUILD_TYPE)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_DEBUG})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_RELEASE})
endif()

SET(CMAKE_CXX_FLAGS "${CXX_COMMON_FLAGS} ${CMAKE_CXX_FLAGS}")

# Given a package name and the version number extracted from the environment, construct
# the package_ROOT variable.
function(set_dep_root NAME)
  string(TOLOWER ${NAME} NAME_LOWER)
  string(REPLACE "_" "-" NAME_LOWER ${NAME_LOWER})
  set(VAL_NAME "IMPALA_${NAME}_VERSION")
  set(${NAME}_ROOT $ENV{IMPALA_TOOLCHAIN_PACKAGES_HOME}/${NAME_LOWER}-$ENV{${VAL_NAME}} PARENT_SCOPE)
endfunction()

set(ENV{BOOST_ROOT} "$ENV{IMPALA_TOOLCHAIN_PACKAGES_HOME}/boost-$ENV{IMPALA_BOOST_VERSION}")
set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME ON)
set_dep_root(GFLAGS)
set_dep_root(GLOG)
set_dep_root(PROTOBUF)
set_dep_root(RAPIDJSON)
set_dep_root(THRIFT)
set_dep_root(GTEST)

# find boost headers and libs
IF (DEFINED ENV{BOOST_ROOT})
  # The casing and underscoring expected for these properties varies between
  # versions of CMake. Multiple inconsistent versions may be present here
  # intentionally to provide what a wide range of versions expects.
  message(STATUS "Using boost from: " $ENV{BOOST_ROOT})
  set(Boost_NO_SYSTEM_PATHS true)
  set(BOOST_ROOT $ENV{BOOST_ROOT})
  set(BOOST_LIBRARYDIR $ENV{BOOST_ROOT}/lib)
  set(BOOST_INCLUDEDIR $ENV{BOOST_ROOT}/include)
  set(Boost_INCLUDE_DIR ${BOOST_INCLUDEDIR})
ENDIF (DEFINED ENV{BOOST_ROOT})

# Locate the LZO compression library.
find_package(Lzo REQUIRED)

# Locate the Thrift headers.
find_package(Thrift REQUIRED)
include_directories(${THRIFT_INCLUDE_DIR})

# find rapidjson headers
find_package(RapidJson REQUIRED)
include_directories(${RAPIDJSON_INCLUDE_DIR})
message(STATUS "RapidJson include dir: " ${RAPIDJSON_INCLUDE_DIR})

add_definitions(-DHAVE_INTTYPES_H -DHAVE_NETINET_IN_H)
add_definitions(-DKUDU_HEADERS_NO_STUBS)

if (CMAKE_SYSTEM_NAME MATCHES "Linux" AND CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
  add_definitions(-DCACHELINESIZE_AARCH64=${CACHELINESIZE_AARCH64})
endif()

# where to put generated libraries
set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build")
set(LIBRARY_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}")

include_directories(${LZO_INCLUDE_DIR})
include_directories($ENV{IMPALA_HOME}/be/src)
include_directories($ENV{IMPALA_HOME}/be/generated-sources)

find_package(GLog REQUIRED)
include_directories(${GLOG_INCLUDE_DIR})

find_package(GFlags REQUIRED)
include_directories(${GFLAGS_INCLUDE_DIR})

find_package(Protobuf REQUIRED)
include_directories(${PROTOBUF_INCLUDE_DIR})

include_directories($ENV{IMPALA_HOME}/be/src/thirdparty) # Needed for squeasel
include_directories($ENV{HADOOP_INCLUDE_DIR})
include_directories(SYSTEM ${BOOST_INCLUDEDIR})

# Locate and include the gtest headers
find_package(GTest REQUIRED)
include_directories(${GTEST_INCLUDE_DIR})

message(STATUS "LZO_LIB: ${LZO_LIB}")

add_library(impalalzo SHARED
  hdfs-lzo-text-scanner.cc
)

target_link_libraries(impalalzo
  ${LZO_LIB}
)
