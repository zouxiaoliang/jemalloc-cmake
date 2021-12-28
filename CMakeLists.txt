cmake_minimum_required(VERSION 3.5)

project(jemalloc-cmake LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(ExternalProject)

message(STATUS "jemalloc bin path: ${CMAKE_CURRENT_BINARY_DIR}")

add_library(thirdparty_jemalloc_static STATIC IMPORTED GLOBAL)

if("${CMAKE_GENERATOR}" STREQUAL "Unix Makefiles")
    set(make_command "$(MAKE)")
else()
    set(make_command "make")
endif()
set(install_prefix ${CMAKE_CURRENT_BINARY_DIR}/.local)
set(configure_command
    "${CMAKE_COMMAND}" -E env
    "CC=${CMAKE_C_COMPILER}"
    "CXX=${CMAKE_CXX_COMPILER}"
    "AR=${CMAKE_AR}"
    "CXXFLAGS=${CMAKE_CXX_FLAGS} -Wno-ignored-attributes -Wno-unknown-warning-option --sysroot ${CMAKE_SYSROOT}"
    "CFLAGS=${CMAKE_C_FLAGS} -Wno-ignored-attributes -Wno-unknown-warning-option --sysroot ${CMAKE_SYSROOT}"
    perl ./configure
    "--prefix=${install_prefix}"
    )
message(STATUS "configure_command: ${configure_command}")

set(build_command ${make_command} -j2 && make install)

message(STATUS "jemalloc source dir: ${CMAKE_CURRENT_SOURCE_DIR}/jemalloc")

ExternalProject_Add(jemalloc
    DOWNLOAD_COMMAND ""
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/jemalloc
    CONFIGURE_COMMAND ${configure_command}
    BUILD_COMMAND ${build_command}
    BUILD_IN_SOURCE true
    BUILD_ALWAYS true)

set_target_properties(thirdparty_jemalloc_static PROPERTIES IMPORTED_LOCATION
    "${install_prefix}/lib/libjemalloc.a")
add_dependencies(thirdparty_jemalloc_static jemalloc)

add_library(thirdparty_facebook_jemalloc INTERFACE)
target_link_libraries(thirdparty_facebook_jemalloc INTERFACE
    thirdparty_jemalloc_static rt pthread)
target_include_directories(thirdparty_facebook_jemalloc INTERFACE ${install_prefix}/include/)
