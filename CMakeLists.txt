cmake_minimum_required(VERSION 2.8.8)
project(instr)

find_package(LLVM REQUIRED CONFIG)
if (${LLVM_PACKAGE_VERSION} VERSION_GREATER "3.4")
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${LLVM_DIR}")
  include(LLVMConfig)
endif()
include(AddLLVM)

# LLVM 10 and newer require at least C++14 standard
if (${LLVM_PACKAGE_VERSION} VERSION_GREATER "9.0")
  set(CMAKE_CXX_STANDARD 14)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
else()
  # otherwise we need C++11 standard
  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()
set(CMAKE_CXX_STANDARD_REQUIRED on)

# Compiler Options
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -Wall -Wextra -Wno-unused-parameter -Wno-unused-variable -fno-rtti")

add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})
link_directories(${LLVM_LIBRARY_DIRS})

include_directories(include)
include_directories(analyses)

find_path(LLVM_SRC_PATH include/llvm/Module.h)
find_path(LLVM_BUILD_PATH include/llvm/Intrinsics.gen)

if (LLVM_SRC_PATH)
  include_directories(${LLVM_SRC_PATH}/include)
  SET(LLVM_INCLUDE_DIRS ${LLVM_SRC_PATH}/include)
  message(STATUS "Got LLVM_SRC_PATH ${LLVM_SRC_PATH}")
endif(LLVM_SRC_PATH)
if (LLVM_BUILD_PATH)
  include_directories(${LLVM_BUILD_PATH}/include)
  link_directories(${LLVM_BUILD_PATH}/lib)
  SET(LLVM_LIBRARY_DIRS ${LLVM_BUILD_PATH}/lib)
  message(STATUS "Got LLVM_BUILD_PATH ${LLVM_BUILD_PATH}")
endif(LLVM_BUILD_PATH)

message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")
message(STATUS "LLVM include dir: ${LLVM_INCLUDE_DIRS}")
message(STATUS "LLVM libraries dir: ${LLVM_LIBRARY_DIRS}")
message(STATUS "LLVM definitions: ${LLVM_DEFINITIONS}")

include_directories(${CMAKE_SOURCE_DIR}/include)
include_directories(${CMAKE_SOURCE_DIR}/lib)
include_directories(${CMAKE_SOURCE_DIR}/analyses)
add_definitions(-DHAVE_LLVM)

if (DG_PATH)
	include_directories(${DG_PATH}/include/)
else()
	include_directories(${CMAKE_SOURCE_DIR}/dg/include)
    set(DG_PATH ${CMAKE_SOURCE_DIR}/dg)
endif()

find_library(LIBLP dgllvmpta PATHS ${DG_PATH}/lib/)
find_library(LIBP dgpta PATHS ${DG_PATH}/lib/)
find_library(LIBDGA dganalysis PATHS ${DG_PATH}/lib/)

if (LIBLP AND LIBP AND LIBDGA)
	message(STATUS "Dg libraries found: ${LIBLP}, ${LIBP}, ${LIBDGA}")
	# those libraries we'll use. We need to use whole LLVMdg,
	# since we'll use the LLVM points-to analysis
	set(dg_libs ${LIBLP} ${LIBP} ${LIBDGA})
else()
	message(FATAL_ERROR "Dg not found! Try specifying -DDG_PATH=<path>.")
endif()

include(GNUInstallDirs)
message(STATUS "CMAKE_INSTALL_LIBDIR: \"${CMAKE_INSTALL_LIBDIR}\"")
message(STATUS "CMAKE_INSTALL_DATADIR: \"${CMAKE_INSTALL_FULL_DATADIR}\"")

include(FindPkgConfig)
pkg_search_module(JSON jsoncpp)
set(JSONCPP_MIN_VERSION "1.9.1")
set(JSONCPP_MAX_VERSION "2")
if (JSON_FOUND AND (JSON_VERSION VERSION_GREATER JSONCPP_MIN_VERSION) AND (JSON_VERSION VERSION_LESS JSONCPP_MAX_VERSION))
	set(JSON_LIBS ${JSON_LINK_LIBRARIES})
	message(STATUS "Using system jsoncpp library")
else()
	set(JSON_FILES jsoncpp.cpp)
    if (JSON_FOUND)
      message(STATUS "System contains jsoncpp library version ${JSON_VERSION}, which is not compatible. Only versions between ${JSONCPP_MIN_VERSION} < v < ${JSONCPP_MAX_VERSION} are compatible.")
    endif()
endif()

option(LLVM_LINK_DYLIB "Link with LLVM dynamically" ON)
if (LLVM_LINK_DYLIB)
	message(STATUS "LLVM linking: dynamic")
	if (${LLVM_PACKAGE_VERSION} VERSION_LESS "3.8")
		set(llvm_libs LLVM-${LLVM_PACKAGE_VERSION})
	else()
		# only LLVM 3.8+ provide unversioned library
		set(llvm_libs LLVM)
	endif()
else()
	# Find the libraries that correspond to the LLVM components
	# that we wish to use
	if (${LLVM_PACKAGE_VERSION} VERSION_GREATER "3.4")
		llvm_map_components_to_libnames(llvm_libs support core irreader bitwriter linker)
	else()
		llvm_map_components_to_libraries(llvm_libs support core irreader bitwriter linker)
	endif()
endif()

enable_testing()

add_subdirectory(analyses)
add_subdirectory(instrumentations)
add_subdirectory(src)
add_subdirectory(tests)
