
cmake_minimum_required(VERSION 2.8.11 FATAL_ERROR)

#version variables, major and minor must be 1 character only
SET(LICENSECC_MAJOR_VERSION 1)
SET(LICENSECC_MINOR_VERSION 1)
SET(LICENSECC_PATCH_VERSION 0 CACHE STRING "License++ patch version string")
SET(LICENSECC_INT_VERSION "${LICENSECC_MAJOR_VERSION}${LICENSECC_MINOR_VERSION}${LICENSECC_PATCH_VERSION}")
SET(LICENSECC_VERSION "${LICENSECC_MAJOR_VERSION}.${LICENSECC_MINOR_VERSION}.${LICENSECC_PATCH_VERSION}")
SET(LICENSECC_SHORT_LICENSE "BSD Software License")

SET(CMAKE_DISABLE_SOURCE_CHANGES ON)
SET(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
SET(CMAKE_VERBOSE_MAKEFILE ON CACHE BOOL "CMake verbose")

project (license++ C CXX )
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/modules/")

include(utilities)
if(CMAKE_BUILD_TYPE)
	string(TOLOWER ${CMAKE_BUILD_TYPE} _CMAKE_BUILD_TYPE)
	EVAL_CONDITION(RELEASE_BUILD ${_CMAKE_BUILD_TYPE} STREQUAL "release")
else(CMAKE_BUILD_TYPE)
	SET(RELEASE_BUILD 0)
endif(CMAKE_BUILD_TYPE)

if(MSVC)
	SET(PLATFORM_LIBS "")
	include(target_arch)
	target_architecture( TARGET_ARCHITECTURE )
	message(STATUS "architecture detected: ${TARGET_ARCHITECTURE}")
	
	#Boost > 3.15 handle the /MD flag more nicely than this
	if(${STATIC_RUNTIME})
		string(REPLACE "/MD" "/MT" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
		string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
		string(REPLACE "/MDd" "/MTd" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
		string(REPLACE "/MDd" "/MTd" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
		SET(Boost_USE_STATIC_RUNTIME ON)
	endif(${STATIC_RUNTIME})

	#SET(CMAKE_FIND_LIBRARY_SUFFIXES .lib ) #${CMAKE_FIND_LIBRARY_SUFFIXES}
	add_definitions("/D _CRT_SECURE_NO_WARNINGS")
else(MSVC)
	#GCC or Mingw
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pipe -fmessage-length=0 -std=c++11 -Wuninitialized -fPIC") #-Wall
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pipe -fmessage-length=0 -Wall -Wuninitialized -fPIC") 
    SET(CMAKE_EXE_LINKER_FLAGS  "-pthread -static-libstdc++")
    SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "-Wl,--strip-all -static-libstdc++") #-static 
    #you need to "force" the change in cmake_install_prefix after the project keyword
	IF(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
		SET(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "Install path prefix" FORCE)
	ENDIF(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    if(MINGW)
        list(APPEND EXTERNAL_LIBS "-lcrypt32 -lws2_32 -liphlpapi")
        SET(CMAKE_EXE_LINKER_FLAGS "-static")
    endif(MINGW)
	
	if(USE_DBUS_IDENTIFIER)
	    FIND_PACKAGE(Dbus REQUIRED)
	    add_definitions(-DUSE_DBUS)
	    include_directories(${DBUS_INCLUDE_DIR})
	    include_directories(${DBUS_ARCH_INCLUDE_DIR})
	    list(APPEND EXTERNAL_LIBS ${DBUS_LIBRARIES})
	endif(USE_DBUS_IDENTIFIER)    
endif(MSVC)

if(UNIX) #this is true for all the linux systems but not for cross compiling
	#find a static version of openssl crypto library
	SET ( OPENSSL_USE_STATIC_LIBS ON )
	find_package(OpenSSL REQUIRED COMPONENTS Crypto)
	include_directories(${OPENSSL_INCLUDE_DIR})
    list(APPEND EXTERNAL_LIBS ${OPENSSL_CRYPTO_LIBRARY})
	MESSAGE(STATUS "Found openssl version ${OPENSSL_VERSION} ")
	#Zlib required when openssl version < 1.0.1f
	SET ( ZLIB_USE_STATIC_LIBS ON )
	find_package(ZLIB REQUIRED)
	list(APPEND EXTERNAL_LIBS ${ZLIB_LIBRARIES})
	MESSAGE(STATUS "Found zlib version ${ZLIB_VERSION} ")
	if(NOT MINGW)
 		list(APPEND EXTERNAL_LIBS "-ldl") 
	endif(NOT MINGW)
	find_program( MEMORYCHECK_COMMAND valgrind )
	set( MEMORYCHECK_COMMAND_OPTIONS "--trace-children=yes --leak-check=full" )
endif(UNIX)

SET_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS $<$<CONFIG:Debug>:_DEBUG>)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  SET(HAVE_64BIT_SIZE_T 1)
else(CMAKE_SIZEOF_VOID_P EQUAL 8)
  SET(HAVE_64BIT_SIZE_T 0)
endif(CMAKE_SIZEOF_VOID_P EQUAL 8)

#bug in cmake win32 - can't find boost compiled with mingw 
if(WIN32 AND "x${CMAKE_CXX_COMPILER_ARCHITECTURE_ID}" STREQUAL "x")
    message(WARNING "WIN32 compiler does not specify CMAKE_CXX_COMPILER_ARCHITECTURE_ID -- filling in manually")
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(CMAKE_CXX_COMPILER_ARCHITECTURE_ID "x64")
    else()
        set(CMAKE_CXX_COMPILER_ARCHITECTURE_ID "x86")
    endif()
    message(STATUS "Compiler architecture: ${CMAKE_CXX_COMPILER_ARCHITECTURE_ID}")
endif()

SET( Boost_USE_STATIC_LIBS ON )
find_package(Boost REQUIRED COMPONENTS date_time unit_test_framework program_options system filesystem)
#set below in case of dynamic linking in debug.
#set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS $<$<CONFIG:Debug>:BOOST_TEST_DYN_LINK>)
add_definitions(-DBOOST_ALL_NO_LIB) #Disable Boost Microsoft magic, all dependencies are handled by cmake
add_definitions(-DBOOST_LIB_DIAGNOSTIC) #Check it is really disabled

configure_file (
	"src/build_properties.h.in" 
	"${CMAKE_BINARY_DIR}/build_properties.h"
)

message( STATUS "C compiler        : " ${CMAKE_C_COMPILER})
message( STATUS "C compiler flags  : " ${CMAKE_C_FLAGS})
message( STATUS "CXX compiler      : " ${CMAKE_CXX_COMPILER})
message( STATUS "CXX compiler flags: " ${CMAKE_CXX_FLAGS})
message( STATUS "CXX compiler flags debug : " ${CMAKE_CXX_FLAGS_DEBUG})
message( STATUS "CXX compiler flags release: " ${CMAKE_CXX_FLAGS_RELEASE})
if(CMAKE_BUILD_TYPE)
  message( STATUS "Build type        : " ${CMAKE_BUILD_TYPE})
endif(CMAKE_BUILD_TYPE)
message( STATUS "Install prefix    : " ${CMAKE_INSTALL_PREFIX})

#include build directory to find build_properties.h
include_directories(${CMAKE_BINARY_DIR})

add_subdirectory(src)

INCLUDE(CTest)
ENABLE_TESTING()
IF(BUILD_TESTING)
  SET(BUILDNAME "${BUILDNAME}" CACHE STRING "Name of build on the dashboard")
  MARK_AS_ADVANCED(BUILDNAME)
ENDIF(BUILD_TESTING)
add_subdirectory(test)


