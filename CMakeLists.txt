cmake_minimum_required(VERSION 3.11)

project(deconz_cli_plugin LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

if (NOT QT_VERSION_MAJOR)
    find_package(QT NAMES Qt6 Qt5 COMPONENTS Core Network REQUIRED)
endif()

find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Core Network REQUIRED)

# --- deCONZ library (headers + sources pulled from GitHub) -------------------
include(FetchContent)

set(DECONZ_LIB_GIT_REPOSITORY "https://github.com/dresden-elektronik/deconz-lib.git"
    CACHE STRING "deCONZ library repository")
set(DECONZ_LIB_GIT_TAG "main"
    CACHE STRING "deCONZ library branch, tag or commit")

FetchContent_GetProperties(deconzlib)
if (NOT deconzlib_POPULATED)
    FetchContent_Populate(deconzlib
        GIT_REPOSITORY ${DECONZ_LIB_GIT_REPOSITORY}
        GIT_TAG        ${DECONZ_LIB_GIT_TAG})

    # Silence warnings from third-party code (deconz-lib, actor_model); only applies to this subtree.
    set(_deconz_saved_c_flags   "${CMAKE_C_FLAGS}")
    set(_deconz_saved_cxx_flags "${CMAKE_CXX_FLAGS}")
    set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -w")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -w")
    add_subdirectory(${deconzlib_SOURCE_DIR} ${deconzlib_BINARY_DIR})
    set(CMAKE_C_FLAGS   "${_deconz_saved_c_flags}")
    set(CMAKE_CXX_FLAGS "${_deconz_saved_cxx_flags}")
endif()

# --- plugin ------------------------------------------------------------------
add_library(${PROJECT_NAME} MODULE
    deconz_cli_plugin.cpp
    deconz_cli_plugin.h
)

target_compile_options(${PROJECT_NAME} PRIVATE -Wno-attributes)

target_link_libraries(${PROJECT_NAME}
    PRIVATE deCONZLib
    PRIVATE Qt${QT_VERSION_MAJOR}::Core
    PRIVATE Qt${QT_VERSION_MAJOR}::Network
)
