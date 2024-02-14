# If Lua is installed in a non-standard location, please set the LUA_DIR
# environment variable to point to prefix for the install. Eg:
#       Unix: export LUA_DIR=/home/user/pkg
#       Windows: set LUA_DIR=c:\lua51

cmake_minimum_required(VERSION 3.11)
project(lua-cjson C)

option(USE_INTERNAL_FPCONV "Use internal strtod() / g_fmt() code for performance")
option(MULTIPLE_THREADS "Support multi-threaded apps with internal fpconv - recommended" ON)
option(ENABLE_CJSON_GLOBAL "Export CJSON to global" OFF)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
        FORCE)
endif()

set(LUA_LIBRARY_TARGET "" CACHE STRING "Specific external Lua library for linking against")

if("${LUA_LIBRARY_TARGET}" STREQUAL "")
    set(CPM_DOWNLOAD_LOCATION "${CMAKE_BINARY_DIR}/cmake/CPM.cmake")
    if(NOT (EXISTS ${CPM_DOWNLOAD_LOCATION}))
        file(DOWNLOAD https://github.com/cpm-cmake/CPM.cmake/releases/download/v0.38.3/CPM.cmake ${CPM_DOWNLOAD_LOCATION}
                EXPECTED_HASH SHA256=cc155ce02e7945e7b8967ddfaff0b050e958a723ef7aad3766d368940cb15494)
    endif()
    include(${CPM_DOWNLOAD_LOCATION})

    CPMAddPackage(
        NAME lua51
        GITHUB_REPOSITORY GameDevDeps/lua51
        GIT_TAG lua/5.1.5)

    find_package(lua51 REQUIRED)

    set(LUA_LIBRARY_TARGET lua51.liblua-static)
endif()

if(ENABLE_CJSON_GLOBAL)
    add_definitions(-DENABLE_CJSON_GLOBAL)
endif()

if(NOT USE_INTERNAL_FPCONV)
    # Use libc number conversion routines (strtod(), sprintf())
    set(FPCONV_SOURCES fpconv.c)
else()
    # Use internal number conversion routines
    add_definitions(-DUSE_INTERNAL_FPCONV)
    set(FPCONV_SOURCES g_fmt.c dtoa.c)

    include(TestBigEndian)
    TEST_BIG_ENDIAN(IEEE_BIG_ENDIAN)
    if(IEEE_BIG_ENDIAN)
        add_definitions(-DIEEE_BIG_ENDIAN)
    endif()

    if(MULTIPLE_THREADS)
        set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
        find_package(Threads REQUIRED)
        if(NOT CMAKE_USE_PTHREADS_INIT)
            message(FATAL_ERROR
                    "Pthreads not found - required by MULTIPLE_THREADS option")
        endif()
        add_definitions(-DMULTIPLE_THREADS)
    endif()
endif()

# Handle platforms missing isinf() macro (Eg, some Solaris systems).
include(CheckSymbolExists)
CHECK_SYMBOL_EXISTS(isinf math.h HAVE_ISINF)
if(NOT HAVE_ISINF)
    add_definitions(-DUSE_INTERNAL_ISINF)
endif()

set(_MODULE_LINK "${CMAKE_THREAD_LIBS_INIT}" "${LUA_LIBRARY_TARGET}")
#get_filename_component(_lua_lib_dir ${LUA_LIBRARY} PATH)

if(APPLE)
    set(CMAKE_SHARED_MODULE_CREATE_C_FLAGS
        "${CMAKE_SHARED_MODULE_CREATE_C_FLAGS} -undefined dynamic_lookup")
endif()

if(WIN32)
    # Win32 modules need to be linked to the Lua library.
#    set(_MODULE_LINK ${LUA_LIBRARY} ${_MODULE_LINK})
#    set(_lua_module_dir "${_lua_lib_dir}")
    # Windows sprintf()/strtod() handle NaN/inf differently. Not supported.
    add_definitions(-DDISABLE_INVALID_NUMBERS)
else()
#    set(_lua_module_dir "${_lua_lib_dir}/lua/5.1")
endif()

add_library(luacjson.cjson STATIC lua_cjson.c strbuf.c ${FPCONV_SOURCES})
add_library(luacjson::cjson ALIAS luacjson.cjson)
set_target_properties(luacjson.cjson PROPERTIES PREFIX "" EXPORT_NAME cjson OUTPUT_NAME cjson)
target_link_libraries(luacjson.cjson PRIVATE ${_MODULE_LINK})
#install(TARGETS cjson DESTINATION "${_lua_module_dir}")

# vi:ai et sw=4 ts=4:
