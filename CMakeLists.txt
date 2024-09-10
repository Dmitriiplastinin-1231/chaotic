cmake_minimum_required(VERSION 3.14)
project(userver-samples-chaotic_service CXX)


# Adding userver dependency
find_package(userver COMPONENTS core chaotic postgresql QUIET)
if(NOT userver_FOUND)  # Fallback to subdirectory usage
    # Compatibility mode: some systems don't support these features
    set(USERVER_FEATURE_CRYPTOPP_BLAKE2 OFF CACHE BOOL "" FORCE)
    set(USERVER_FEATURE_GRPC_CHANNELZ OFF CACHE BOOL "" FORCE)
    set(USERVER_FEATURE_REDIS_HI_MALLOC ON CACHE BOOL "" FORCE)

    if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/third_party/userver)
        message(STATUS "Using userver framework from third_party/userver")
        add_subdirectory(third_party/userver)
    else()
        message(FATAL_ERROR "Either install the userver or provide a path to it")
    endif()
endif()

userver_setup_environment()

# /// [objects]
add_library(${PROJECT_NAME}_objs OBJECT
    src/say_hello.hpp
    src/say_hello.cpp
    src/hello_service.hpp
    src/hello_service.cpp
)
target_link_libraries(${PROJECT_NAME}_objs userver::core)
target_include_directories(${PROJECT_NAME}_objs PUBLIC src)
# /// [objects]

# /// [chaotic]
file(GLOB_RECURSE SCHEMAS ${CMAKE_CURRENT_SOURCE_DIR}/schemas/*.yaml)
userver_target_generate_chaotic(${PROJECT_NAME}-chgen
    ARGS
        # Map '/components/schemas/*' JSONSchema types to C++ types in 'samples::hello' namespace
        -n "/components/schemas/([^/]*)/=samples::hello::{0}"
        -f "(.*)={0}"
        # Don't call clang-format
        --clang-format=
        # Generate serializers for responses
        --generate-serializers
    OUTPUT_DIR
        ${CMAKE_CURRENT_BINARY_DIR}/src
    SCHEMAS
        ${SCHEMAS}
    RELATIVE_TO
        ${CMAKE_CURRENT_SOURCE_DIR}
)
target_link_libraries(${PROJECT_NAME}_objs ${PROJECT_NAME}-chgen)
# /// [chaotic]


# /// [executable]
add_executable(${PROJECT_NAME} main.cpp)
target_link_libraries(${PROJECT_NAME} ${PROJECT_NAME}_objs)
# /// [executable]

# /// [unittests]
add_executable(${PROJECT_NAME}-unittest unittests/say_hello_test.cpp)
target_link_libraries(${PROJECT_NAME}-unittest ${PROJECT_NAME}_objs userver::utest)
add_google_tests(${PROJECT_NAME}-unittest)
# /// [unittests]

# /// [benchmarks]
add_executable(${PROJECT_NAME}_benchmark benchmarks/say_hello_bench.cpp)
target_link_libraries(${PROJECT_NAME}_benchmark ${PROJECT_NAME}_objs userver::ubench)
add_google_benchmark_tests(${PROJECT_NAME}_benchmark)
# /// [benchmarks]

# /// [testsuite]
# Makes a virtualenv suitable for running pytests from `test` directory and integrates with `ctest`.
userver_testsuite_add_simple()
# /// [testsuite]
