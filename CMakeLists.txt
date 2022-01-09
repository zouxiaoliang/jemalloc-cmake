cmake_minimum_required(VERSION 3.5)

project(jemalloc-cmake LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(ExternalProject)

message(STATUS "jemalloc bin path: ${CMAKE_CURRENT_BINARY_DIR}")

function(jemallocMain)
    add_library(thirdparty_jemalloc_static STATIC IMPORTED GLOBAL)

    if("${CMAKE_GENERATOR}" STREQUAL "Unix Makefiles")
        set(make_command "$(MAKE)")
    else()
        set(make_command "make")
    endif()
    set(install_prefix ${CMAKE_CURRENT_BINARY_DIR}/.local)

    if (CMAKE_SYSROOT)
        set (FLAGS
            "CXXFLAGS=-Wno-ignored-attributes -Wno-unknown-warning-option ${CMAKE_CXX_FLAGS} --sysroot ${CMAKE_SYSROOT}"
            "CFLAGS=-Wno-ignored-attributes -Wno-unknown-warning-option ${CMAKE_C_FLAGS} --sysroot ${CMAKE_SYSROOT}"
            )
    else()
        set (FLAGS
            "CXXFLAGS=-Wno-ignored-attributes -Wno-unknown-warning-option ${CMAKE_CXX_FLAGS}"
            "CFLAGS=-Wno-ignored-attributes -Wno-unknown-warning-option ${CMAKE_C_FLAGS}"
            )
    endif()

    set(configure_command
        "${CMAKE_COMMAND}" -E env
        "CC=${CMAKE_C_COMPILER}"
        "CXX=${CMAKE_CXX_COMPILER}"
        "AR=${CMAKE_AR}"
        "${FLAGS}"
        perl ./configure
        "--disable-debug"
        "--enable-prof=true"
        "--prefix=${install_prefix}"
        "--exec-prefix=${install_prefix}"
        )
    message(STATUS "configure_command: ${configure_command}")

    set(build_command ${make_command} -j2 && make install)

    message(STATUS "jemalloc source dir: ${CMAKE_CURRENT_SOURCE_DIR}/jemalloc")

    find_program(AUTOCONF_BIN "CC=${CMAKE_C_COMPILER}"
        "CXX=${CMAKE_CXX_COMPILER}"
        "AR=${CMAKE_AR}" autoconf PATHS /opt/homebrew/bin/ /bin/ /usr/bin/ /usr/local/bin)
    message(STATUS "autoconf path: ${AUTOCONF_BIN}")
    execute_process(COMMAND  ${AUTOCONF_BIN} WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/jemalloc OUTPUT_VARIABLE JEMALLOC_AUTOCONF_OUTPUT ERROR_VARIABLE JEMALLOC_AUTOCONF_ERROR)

    if (JEMALLOC_AUTOCONF_ERROR)
        message(STATUS "jemalloc autoconf error, waht: ${JEMALLOC_AUTOCONF_ERROR}")
    endif()
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

    add_library(thirdparty_jemalloc INTERFACE)
    target_link_libraries(thirdparty_jemalloc INTERFACE
        thirdparty_jemalloc_static rt pthread)
    target_include_directories(thirdparty_jemalloc INTERFACE ${install_prefix}/include/)
    target_compile_definitions(thirdparty_jemalloc INTERFACE USING_JEMALLOC=1)

endfunction()

jemallocMain()
