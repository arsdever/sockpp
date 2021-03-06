# CMakeLists.txt
#
# Top-level CMake build file for the 'sockpp' library.
#
# ---------------------------------------------------------------------------
# This file is part of the "sockpp" C++ socket library.
#
# Copyright (c) 2017-2018 Frank Pagliughi
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are
# met:
#
# 1. Redistributions of source code must retain the above copyright notice,
# this list of conditions and the following disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright
# notice, this list of conditions and the following disclaimer in the
# documentation and/or other materials provided with the distribution.
#
# 3. Neither the name of the copyright holder nor the names of its
# contributors may be used to endorse or promote products derived from this
# software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS
# IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO,
# THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
# LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
# NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
# SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
# ---------------------------------------------------------------------------

# --- CMake required version ---
cmake_minimum_required(VERSION 3.5)

# --- Project wide names ---

set(SOCKPP sockpp)
set(SOCKPP_NAMESPACE ${SOCKPP}::)

# --- Project setup ---
project(${SOCKPP} VERSION "0.7.0")

# --- Build Options ---

if(WIN32)
	option(SOCKPP_BUILD_STATIC "Build static library" ON)
	option(SOCKPP_BUILD_SHARED "Build shared library (DLL)" OFF)
	include(GenerateExportHeader)
else()
	option(SOCKPP_BUILD_STATIC "Build static library" OFF)
	option(SOCKPP_BUILD_SHARED "Build shared library" ON)
endif()

option(SOCKPP_BUILD_EXAMPLES "Build example applications" OFF)
option(SOCKPP_BUILD_TESTS "Build unit tests" OFF)
option(SOCKPP_BUILD_DOCUMENTATION "Create Doxygen reference documentation" OFF)

# --- Setting naming variables ---

set(SOCKPP_SHARED_LIBRARY ${SOCKPP})
set(SOCKPP_STATIC_LIBRARY ${SOCKPP}-static)
set(SOCKPP_OBJECT_LIBRARY ${SOCKPP}-objs)
set(SOCKPP_INCLUDES ${CMAKE_SOURCE_DIR}/include)
set(SOCKPP_GENERATED ${CMAKE_BINARY_DIR}/generated)

# --- C++14 build flags ---

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

# --- Generate a version header ---

configure_file(
	${CMAKE_SOURCE_DIR}/cmake/version.h.in  
	${CMAKE_BINARY_DIR}/generated/${SOCKPP}/version.h
	@ONLY
)

# --- Common Library Sources, etc ---

add_subdirectory(src)

# --- Header Locations ---

target_include_directories(${SOCKPP_OBJECT_LIBRARY}
	PUBLIC
		$<BUILD_INTERFACE:${SOCKPP_INCLUDES}>
		$<INSTALL_INTERFACE:include>
	PRIVATE 
		${SOCKPP_GENERATED}
		${CMAKE_SOURCE_DIR}/src
)

# --- System libraries ---

include(GNUInstallDirs)

if(WIN32)
	set(LIBS_SYSTEM ws2_32)
elseif(UNIX)
	set(LIBS_SYSTEM c stdc++)
endif()

# --- Collect the targets names ---

if(${SOCKPP_BUILD_SHARED})
	list(APPEND SOCKPP_TARGETS ${SOCKPP_SHARED_LIBRARY})
endif()

if(${SOCKPP_BUILD_STATIC})
	list(APPEND SOCKPP_TARGETS ${SOCKPP_STATIC_LIBRARY})
endif()

# --- Create the libraries and exporting them ---

if(NOT SOCKPP_TARGETS)
	message(FATAL_ERROR "No targets are specified")
else()
	if(${SOCKPP_BUILD_SHARED})
		message(STATUS "Creating shared library: ${SOCKPP_SHARED_LIBRARY}")
		add_library(${SOCKPP_SHARED_LIBRARY} SHARED $<TARGET_OBJECTS:${SOCKPP_OBJECT_LIBRARY}>)
		target_include_directories(${SOCKPP_SHARED_LIBRARY}
			PUBLIC
				$<BUILD_INTERFACE:${SOCKPP_INCLUDES}>
				$<INSTALL_INTERFACE:include>
			PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/generated)
		target_link_libraries(${SOCKPP_SHARED_LIBRARY} ${LIBS_SYSTEM})
		
		set_target_properties(${SOCKPP_SHARED_LIBRARY} PROPERTIES
			VERSION ${PROJECT_VERSION}
			SOVERSION ${PROJECT_VERSION_MAJOR})
		list(APPEND TARGET_FILES ${SOCKPP_SHARED_LIBRARY})
	endif()
	
	if(${SOCKPP_BUILD_STATIC})
		message(STATUS "Creating static library: ${SOCKPP_STATIC_LIBRARY}")
		add_library(${SOCKPP_STATIC_LIBRARY} STATIC $<TARGET_OBJECTS:${SOCKPP_OBJECT_LIBRARY}>)
		target_include_directories(${SOCKPP_STATIC_LIBRARY}
			PUBLIC
				$<BUILD_INTERFACE:${SOCKPP_INCLUDES}>
				$<INSTALL_INTERFACE:include>
			PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/generated)
		target_link_libraries(${SOCKPP_STATIC_LIBRARY} ${LIBS_SYSTEM})
		
		set_target_properties(${SOCKPP_STATIC_LIBRARY} PROPERTIES
			VERSION ${PROJECT_VERSION}
			SOVERSION ${PROJECT_VERSION_MAJOR})
	
		if(UNIX)
			set_target_properties(${SOCKPP_STATIC_LIBRARY} PROPERTIES OUTPUT_NAME ${SOCKPP_SHARED_LIBRARY})
		endif()
		
		list(APPEND TARGET_FILES ${SOCKPP_STATIC_LIBRARY})
	endif()
		
	install(TARGETS ${TARGET_FILES}
		EXPORT sockppConfig
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
	
	export(TARGETS ${SOCKPP_TARGETS}
		NAMESPACE ${SOCKPP_NAMESPACE}
		FILE "${CMAKE_BINARY_DIR}/sockppConfig.cmake"
	)
	
	install(EXPORT
			sockppConfig
		DESTINATION "${CMAKE_INSTALL_DATADIR}/sockpp/cmake"
		NAMESPACE ${SOCKPP_NAMESPACE}
	)
endif()

# --- Documentation ---

if(SOCKPP_BUILD_DOCUMENTATION)
	add_subdirectory(doc)
endif()

# --- Default library for examples and unit tests ---

if(SOCKPP_BUILD_SHARED)
	set(SOCKPP_LIB ${SOCKPP_SHARED_LIBRARY})
else()
	set(SOCKPP_LIB ${SOCKPP_STATIC_LIBRARY})
endif()


# --- Examples Applications ---

if(SOCKPP_BUILD_EXAMPLES)
	add_subdirectory(examples/tcp)
	add_subdirectory(examples/udp)
	if(UNIX)
		add_subdirectory(examples/unix)
	endif()
endif()

# --- Unit Tests ---

if(SOCKPP_BUILD_TESTS)
	add_subdirectory(tests/unit)
endif()
