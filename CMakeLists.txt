cmake_minimum_required(VERSION 3.25)
project(mredis
    VERSION 0.1.0
    DESCRIPTION "A Redis-inspired in-memory key-value server"
    LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(BUILD_TESTING "Build unit tests" ON)
option(BUILD_BENCHMARKS "Build benchmarks" ON)
option(ENABLE_SANITIZERS "Enable sanitizers" OFF)

set(SANITIZER "none" CACHE STRING "Sanitizer type: none, address, undefined, thread")
set_property(CACHE SANITIZER PROPERTY STRINGS none address undefined thread)

add_library(mredis_warnings INTERFACE)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(mredis_warnings INTERFACE
        -Wall
        -Wextra
        -Wpedantic
        -Wconversion
        -Wsign-conversion
        -Wshadow
        -Wformat=2
        -Wnull-dereference
        -Wdouble-promotion
        -Wimplicit-fallthrough
        -Woverloaded-virtual
    )
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        target_compile_options(mredis_warnings INTERFACE
            -Wduplicated-cond
            -Wduplicated-branches
            -Wlogical-op
            -Wuseless-cast
        )
    endif()
endif()

if(ENABLE_SANITIZERS)
    set(SANITIZER_FLAGS "")

    if(SANITIZER STREQUAL "address")
        set(SANITIZER_FLAGS -fsanitize=address -fno-omit-frame-pointer)
    elseif(SANITIZER STREQUAL "undefined")
        set(SANITIZER_FLAGS -fsanitize=undefined -fno-omit-frame-pointer)
    elseif(SANITIZER STREQUAL "thread")
        set(SANITIZER_FLAGS -fsanitize=thread)
    elseif(NOT SANITIZER STREQUAL "none")
        message(FATAL_ERROR "Unknown sanitizer: ${SANITIZER}")
    endif()

    if(SANITIZER_FLAGS)
        add_library(mredis_sanitizers INTERFACE)
        target_compile_options(mredis_sanitizers INTERFACE ${SANITIZER_FLAGS})
        target_link_options(mredis_sanitizers INTERFACE ${SANITIZER_FLAGS})
    endif()
endif()

add_library(mredis_core
    src/mredis.cpp
)
target_include_directories(mredis_core PUBLIC
    ${PROJECT_SOURCE_DIR}/include
)
target_link_libraries(mredis_core PRIVATE
    mredis_warnings
)

if(TARGET mredis_sanitizers)
    target_link_libraries(mredis_core PRIVATE mredis_sanitizers)
endif()

add_executable(mredis
    src/main.cpp
)

if(BUILD_TESTING)
    enable_testing()
    find_package(GTest REQUIRED)
    add_executable(mredis_tests
        tests/example.cpp
    )
    target_link_libraries(mredis_tests PRIVATE
        mredis_core
        GTest::gtest_main
        mredis_warnings
    )
    if(TARGET mredis_sanitizers)
        target_link_libraries(mredis_tests PRIVATE mredis_sanitizers)
    endif()
    include(GoogleTest)
    gtest_discover_tests(mredis_tests)
endif()

if(BUILD_BENCHMARKS)
    find_package(benchmark REQUIRED)
    add_executable(mredis_benchmarks
        benchmarks/example.cpp
    )
    target_link_libraries(mredis_benchmarks PRIVATE
        mredis_core
        benchmark::benchmark
        mredis_warnings
    )
    if(TARGET mredis_sanitizers)
        target_link_libraries(mredis_benchmarks PRIVATE mredis_sanitizers)
    endif()
endif()
