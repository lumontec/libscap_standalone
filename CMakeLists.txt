#
# Copyright (C) 2013-2018 Draios Inc dba Sysdig.
#
# This file is part of sysdig .
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#


if(EXISTS ${CMAKE_CURRENT_BINARY_DIR}/CMakeLists.txt)
	message(FATAL_ERROR
		"Looks like you are trying to run cmake from the base sysdig source directory.\n"
		"** RUNNING CMAKE FROM THE BASE SYSDIG DIRECTORY WILL NOT WORK **\n"
		"To Fix:\n"
		" 1. Remove the CMakeCache.txt file in this directory. ex: rm CMakeCache.txt\n"
		" 2. Create a build directory from here. ex: mkdir build\n"
		" 3. cd into that directory. ex: cd build\n"
		" 4. Run cmake from the build directory. ex: cmake ..\n"
		" 5. Run make from the build directory. ex: make\n"
		"Full paste-able example:\n"
		"( rm -f CMakeCache.txt; mkdir build; cd build; cmake ..; make )\n"
		"The following wiki page has more information on manually building sysdig: http://bit.ly/1oJ84UI")
endif()

cmake_minimum_required(VERSION 2.8.2)

project(libscap)

set(PACKAGE_NAME "libscap")

# Add path for custom CMake modules.
list(APPEND CMAKE_MODULE_PATH
	"${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

add_definitions(-DPLATFORM_NAME="${CMAKE_SYSTEM_NAME}")
add_definitions(-DK8S_DISABLE_THREAD)

option(BUILD_WARNINGS_AS_ERRORS "Enable building with -Wextra -Werror flags")

if(MINIMAL_BUILD)
  set(MINIMAL_BUILD_FLAGS "-DMINIMAL_BUILD")
endif()

if(MUSL_OPTIMIZED_BUILD)
	set(SYSDIG_MUSL_FLAGS "-static -Os")
endif()


if(NOT WIN32)

	set(SYSDIG_DEBUG_FLAGS "-D_DEBUG")
	set(CMAKE_COMMON_FLAGS "-Wall -ggdb ${MINIMAL_BUILD_FLAGS} ${SYSDIG_MUSL_FLAGS}")

	if(BUILD_WARNINGS_AS_ERRORS)
		set(CMAKE_SUPPRESSED_WARNINGS "-Wno-unused-parameter -Wno-missing-field-initializers -Wno-sign-compare -Wno-type-limits -Wno-implicit-fallthrough -Wno-format-truncation")
		set(CMAKE_COMMON_FLAGS "${CMAKE_COMMON_FLAGS} -Wextra -Werror ${CMAKE_SUPPRESSED_WARNINGS}")
	endif()

	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_COMMON_FLAGS}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_COMMON_FLAGS} -std=c++0x")

	set(CMAKE_C_FLAGS_DEBUG "${SYSDIG_DEBUG_FLAGS}")
	set(CMAKE_CXX_FLAGS_DEBUG "${SYSDIG_DEBUG_FLAGS}")

	set(CMAKE_C_FLAGS_RELEASE "-O3 -fno-strict-aliasing -DNDEBUG")
	set(CMAKE_CXX_FLAGS_RELEASE "-O3 -fno-strict-aliasing -DNDEBUG")

	if(CMAKE_SYSTEM_NAME MATCHES "Linux")
		if(CMAKE_BUILD_TYPE STREQUAL "Debug")
			set(KBUILD_FLAGS "${SYSDIG_DEBUG_FLAGS} ${SYSDIG_FEATURE_FLAGS}")
		else()
			set(KBUILD_FLAGS "${SYSDIG_FEATURE_FLAGS}")
		endif()

		if(NOT DEFINED PROBE_VERSION)
			set(PROBE_VERSION "${SYSDIG_VERSION}")
		endif()
		if(NOT DEFINED PROBE_NAME)
			set(PROBE_NAME "sysdig-probe")
		endif()

		if(NOT DEFINED PROBE_DEVICE_NAME)
			set(PROBE_DEVICE_NAME "sysdig")
		endif()

		add_subdirectory(driver)
		add_definitions(-DHAS_CAPTURE)
	endif()

	add_subdirectory(scripts)

	if(CMAKE_SYSTEM_NAME MATCHES "SunOS")
		set(CMD_MAKE gmake)
	else()
		set(CMD_MAKE make)
	endif()
endif()

include(ExternalProject)

option(USE_BUNDLED_DEPS "Enable bundled dependencies instead of using the system ones" ON)

if(MINIMAL_BUILD)
  set(MINIMAL_BUILD_FLAGS "-DMINIMAL_BUILD")
endif()

add_definitions(-DPLATFORM_NAME="${CMAKE_SYSTEM_NAME}")

include_directories("${PROJECT_SOURCE_DIR}/common")

option(USE_BUNDLED_ZLIB "Enable building of the bundled zlib" ${USE_BUNDLED_DEPS})

if(NOT MINIMAL_BUILD)
	if(NOT USE_BUNDLED_ZLIB)
		find_path(ZLIB_INCLUDE zlib.h PATH_SUFFIXES zlib)
		find_library(ZLIB_LIB NAMES z)
		if(ZLIB_INCLUDE AND ZLIB_LIB)
			message(STATUS "Found zlib: include: ${ZLIB_INCLUDE}, lib: ${ZLIB_LIB}")
		else()
			message(FATAL_ERROR "Couldn't find system zlib")
		endif()
	else()
		set(ZLIB_SRC "${PROJECT_BINARY_DIR}/zlib-prefix/src/zlib")
		message(STATUS "Using bundled zlib in '${ZLIB_SRC}'")
		set(ZLIB_INCLUDE "${ZLIB_SRC}")
		if(NOT WIN32)
			set(ZLIB_LIB "${ZLIB_SRC}/libz.a")
			ExternalProject_Add(zlib
				URL "http://download.draios.com/dependencies/zlib-1.2.11.tar.gz"
				URL_MD5 "1c9f62f0778697a09d36121ead88e08e"
				CONFIGURE_COMMAND "./configure"
				BUILD_COMMAND ${CMD_MAKE}
				BUILD_IN_SOURCE 1
				BUILD_BYPRODUCTS ${ZLIB_LIB}
				INSTALL_COMMAND "")
		else()
			set(ZLIB_LIB "${ZLIB_SRC}/zdll.lib")
			ExternalProject_Add(zlib
				URL "http://download.draios.com/dependencies/zlib-1.2.11.tar.gz"
				URL_MD5 "1c9f62f0778697a09d36121ead88e08e"
				CONFIGURE_COMMAND ""
				BUILD_COMMAND nmake -f win32/Makefile.msc
				BUILD_IN_SOURCE 1
				BUILD_BYPRODUCTS ${ZLIB_LIB}
				INSTALL_COMMAND "")
		endif()
	endif()
endif() # MINIMAL_BUILD

if(NOT MINIMAL_BUILD)
include_directories("${ZLIB_INCLUDE}")
endif()
if(CYGWIN)
#include_directories("${WIN_HAL_INCLUDE}")
endif()

list(APPEND targetfiles
	scap.c
	scap_event.c
	scap_fds.c
	scap_iflist.c
	scap_savefile.c
	scap_procs.c
	scap_userlist.c
	syscall_info_table.c
	./driver/dynamic_params_table.c
	./driver/event_table.c
	./driver/flags_table.c)

if(NOT APPLE)
	list(APPEND targetfiles
		scap_udig.c)
endif()


if(CMAKE_SYSTEM_NAME MATCHES "Linux")
	list(APPEND targetfiles
		scap_bpf.c
		./driver/syscall_table.c
		./driver/fillers_table.c)

    include_directories(${PROJECT_BINARY_DIR}/driver/src)
endif()

if(CYGWIN)
	list(APPEND targetfiles
		windows_hal.c)
endif()

add_library(scap STATIC
	${targetfiles})

if(USE_BUNDLED_ZLIB AND NOT MINIMAL_BUILD)
	add_dependencies(scap zlib)
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Linux")
	target_link_libraries(scap
		elf
		rt)
endif()

if(NOT MINIMAL_BUILD)
target_link_libraries(scap
	"${ZLIB_LIB}")
endif()

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
    option(BUILD_LIBSCAP_EXAMPLES "Build libscap examples" ON)

    if (BUILD_LIBSCAP_EXAMPLES)
        add_subdirectory(examples/01-open)
        add_subdirectory(examples/02-validatebuffer)
    endif()

	include(FindMakedev)
endif()



