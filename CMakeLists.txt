# Copyright 2020 Jeremiah Griffin
#
# Permission to use, copy, modify, and/or distribute this software for any
# purpose with or without fee is hereby granted, provided that the above
# copyright notice and this permission notice appear in all copies.
#
# THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
# WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
# MERCHANTABILITY AND FITNESS.  IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
# ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
# WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
# ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
# OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.

cmake_minimum_required(VERSION 3.1...3.15)
if(${CMAKE_VERSION} VERSION_LESS 3.12)
    cmake_policy(VERSION ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION})
endif()

# Project-wide
project(npln
    VERSION 0.1.0
    DESCRIPTION "A programming environment for PL/0 on CHIP-8"
    LANGUAGES C CXX
)

include(CTest)

# Helpers
function(disable_clang_tidy target)
    get_target_property(target_type ${target} TYPE)
    get_target_property(target_alias ${target} ALIASED_TARGET)
    if((NOT target_type STREQUAL "INTERFACE_LIBRARY") AND (target_alias STREQUAL ""))
        # Clear the *_CLANG_TIDY properties so that any global setting is
        # discarded.  This prevents clang-tidy from being run on this target.
        set_target_properties(${target}
            PROPERTIES
            C_CLANG_TIDY ""
            CXX_CLANG_TIDY ""
        )
    endif()
endfunction()

# Options
option(BUILD_RUNNER "Build the runner graphical interface" TRUE)
if(BUILD_RUNNER)
    set(BUILD_RENDERER ON)
endif()

# Dependencies
find_package(fmt 6.0.0 REQUIRED)

find_package(CLI11 1.9.0 REQUIRED)

if(BUILD_RUNNER)
    find_package(glfw3 3.3.0 REQUIRED)
endif()

# Vendored dependencies
add_subdirectory(vendor/catch2)
include(vendor/catch2/contrib/Catch.cmake)
disable_clang_tidy(Catch2::Catch2)

add_subdirectory(vendor/gsl)
disable_clang_tidy(Microsoft.GSL::GSL)

add_library(scope_guard INTERFACE)
target_include_directories(scope_guard
    INTERFACE
    vendor/scope_guard
)
disable_clang_tidy(scope_guard)

if(BUILD_RENDERER)
    add_library(glad STATIC
        vendor/glad/include/KHR/khrplatform.h
        vendor/glad/include/glad/glad.h
        vendor/glad/src/glad.c
    )
    target_link_libraries(glad
        PUBLIC
        ${CMAKE_DL_LIBS}
    )
    target_include_directories(glad
        PUBLIC
        vendor/glad/include
    )
    disable_clang_tidy(glad)
endif()

# libnpln target
add_library(libnpln
    libnpln/detail/ContainerCast.hpp
    libnpln/detail/cpp2b.hpp
    libnpln/machine/BitCodec.hpp
    libnpln/machine/BitCodecs.hpp
    libnpln/machine/DataUnits.hpp
    libnpln/machine/Display.cpp
    libnpln/machine/Display.hpp
    libnpln/machine/Fault.hpp
    libnpln/machine/Font.cpp
    libnpln/machine/Font.hpp
    libnpln/machine/Instruction.hpp
    libnpln/machine/Key.hpp
    libnpln/machine/Keys.cpp
    libnpln/machine/Keys.hpp
    libnpln/machine/Machine.cpp
    libnpln/machine/Machine.hpp
    libnpln/machine/Memory.cpp
    libnpln/machine/Memory.hpp
    libnpln/machine/Operand.hpp
    libnpln/machine/Operands.hpp
    libnpln/machine/Operator.hpp
    libnpln/machine/Register.hpp
    libnpln/machine/RegisterRange.hpp
    libnpln/machine/Registers.hpp
    libnpln/machine/Stack.hpp
    libnpln/utility/BitSetDifference.hpp
    libnpln/utility/FixedSizeStack.hpp
    libnpln/utility/HexDump.hpp
    libnpln/utility/Numeric.hpp
)
target_compile_features(libnpln
    PUBLIC
    cxx_std_17
)
target_link_libraries(libnpln
    PUBLIC
    Microsoft.GSL::GSL
    fmt::fmt
)
target_include_directories(libnpln
    PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:>
)

# libnpln tests
add_executable(test-libnpln
    libnpln/libnpln.test.cpp
    libnpln/machine/BitCodec.test.cpp
    libnpln/machine/DataUnits.test.cpp
    libnpln/machine/Display.test.cpp
    libnpln/machine/Instruction.test.cpp
    libnpln/machine/Fault.test.cpp
    libnpln/machine/Font.test.cpp
    libnpln/machine/Key.test.cpp
    libnpln/machine/Keys.test.cpp
    libnpln/machine/Machine.test.cpp
    libnpln/machine/Memory.test.cpp
    libnpln/machine/Operand.test.cpp
    libnpln/machine/Operands.test.cpp
    libnpln/machine/Operator.test.cpp
    libnpln/machine/Register.test.cpp
    libnpln/machine/RegisterRange.test.cpp
    libnpln/machine/Registers.test.cpp
    libnpln/machine/Stack.test.cpp
    libnpln/utility/BitSetDifference.test.cpp
    libnpln/utility/FixedSizeStack.test.cpp
    libnpln/utility/Numeric.test.cpp
)
target_link_libraries(test-libnpln
    libnpln
    Catch2::Catch2
)
catch_discover_tests(test-libnpln)

# npln target
if(BUILD_RUNNER)
    set(BUILD_RENDERER ON)
    set(npln_RUNNER_SOURCE
        npln/runner/Interface.cpp
        npln/runner/Interface.hpp
        npln/runner/Parameters.hpp
        npln/runner/Runner.cpp
        npln/runner/Runner.hpp
    )
    set(npln_RUNNER_LIBRARIES
        glfw
    )
endif()
if(BUILD_RENDERER)
    set(npln_RENDERER_SOURCE
    )
    set(npln_RENDERER_LIBRARIES
        glad
    )
endif()
configure_file(
    npln/Build.hpp.in
    conf/npln/Build.hpp
    @ONLY
)
add_executable(npln
    ${CMAKE_CURRENT_BINARY_DIR}/conf/npln/Build.hpp
    ${npln_RENDERER_SOURCE}
    ${npln_RUNNER_SOURCE}
    npln/main.cpp
)
target_link_libraries(npln
    libnpln
    scope_guard
    CLI11::CLI11
    ${npln_RENDERER_LIBRARIES}
    ${npln_RUNNER_LIBRARIES}
)
target_include_directories(npln
    PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/conf>
)
