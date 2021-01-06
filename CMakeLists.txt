cmake_minimum_required(VERSION 2.8.12)

find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    message(STATUS "Crypto: Found ccache package... Activating...")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

if (DEFINED CMAKE_BUILD_TYPE)
    set (CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "Choose the type of build, options are: Debug, Release, RelWithDebInfo")
else ()
    set (CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: Debug, Release, RelWithDebInfo")
endif ()

set(LIB_MAJOR_VERSION "1")
set(LIB_MINOR_VERSION "0")
set(LIB_PATCH_VERSION "0")
set(LIB_VERSION_STRING "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_PATCH_VERSION}")

if(CMAKE_VERSION VERSION_LESS 3.0)
    project(ed25519-donna C)
    enable_language(C)
else()
    cmake_policy(SET CMP0048 NEW)
    project(ed25519-donna VERSION "${LIB_VERSION_STRING}" LANGUAGES C)
endif()

## This section describes our general CMake setup options
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_SKIP_INSTALL_RULES OFF FORCE)
set(CMAKE_SKIP_PACKAGE_ALL_DEPENDENCY ON FORCE)
set(CMAKE_SUPPRESS_REGENERATION ON)
set(CMAKE_POLICY_WARNING_CMP0048 OFF)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# Enable c++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

## We only build static binaries -- this is left here for our dependencies
set(STATIC ON CACHE BOOL FORCE "Link libraries statically? Forced to ON")
add_definitions(-DSTATICLIB)

option(ED25519_TEST "Use non-thread safe reference CSPRNG" ON)
option(ED25519_REFHASH "Use simple/slow implementation of SHA-512" ON)
option(ED25519_FORCE_32BIT "Force use of 32-bit routines" OFF)
option(ED25519_INLINE_ASM "Disable custom assembler routines and rely on portable C" OFF)
option(ED25519_CUSTOMRANDOM "Use custom random implementation" OFF)
option(ED25519_CUSTOMHASH "Use custom hash implementation" OFF)

if(ED25519_TEST)
    add_definitions(-DED25519_TEST)
endif()

if(ED25519_REFHASH)
    add_definitions(-DED25519_REFHASH)
endif()

if(ED25519_FORCE_32BIT)
    add_definitions(-DED25519_FORCE_32BIT)
endif()

if(ED25519_INLINE_ASM)
    add_definitions(-DED25519_INLINE_ASM)
endif()

if(ED25519_CUSTOMRANDOM)
    add_definitions(-DED25519_CUSTOMRANDOM)
endif()

if(ED25519_CUSTOMHASH)
    add_definitions(-DED25519_CUSTOMHASH)
endif()

if(MSVC)
    add_definitions("/EHsc /DWIN32_LEAN_AND_MEAN /O2")
endif()

if(NOT MSVC)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 -Wno-c99-compat -O3")

    if(NOT ED25519_FORCE_32BIT)
        if(CMAKE_SIZEOF_VOID_P EQUAL 8)
            message(STATUS "Configuring for 64-bits")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m64")
        elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
            message(STATUS "Configuring for 32-bits")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")
        endif()
    endif()
endif()

include_directories(${CMAKE_SOURCE_DIR})

set(SOURCES
    ed25519.c
)

add_library(ed25519-donna STATIC ${SOURCES})

if(WIN32)
    target_link_libraries(ed25519-donna ws2_32)
endif()

foreach(flag_var
        CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
        CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO
        CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
        CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    string(REGEX REPLACE "/RTC[^ ]*" "" ${flag_var} "${${flag_var}}")
endforeach(flag_var)
