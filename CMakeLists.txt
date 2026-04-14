#STATIC CMakeLists.txt for the co project
# This file defines the 'co' library and its dependencies using CMake.

cmake_minimum_required(VERSION 4.2)
project(co LANGUAGES CXX ASM) # Add ASM language for .S files

option(CO_ENABLE_TESTING "Embed features for testing in build" ON)

# --- Apple Silicon (ARM64) specific settings ---
# CMAKE_OSX_ARCHITECTURES is expected to be set by the parent project or environment.
# We will propagate this value to sub-dependencies.
if(APPLE)
    message(STATUS "Building for Apple platform. Propagating CMAKE_OSX_ARCHITECTURES: ${CMAKE_OSX_ARCHITECTURES}")
endif()

# Include FetchContent module to manage Abseil dependency
include(FetchContent)

# --- External Dependency: Abseil ---
# The 'co' project depends on Abseil, so we need to fetch and make it available.
FetchContent_Declare(
    abseil
    GIT_REPOSITORY https://github.com/abseil/abseil-cpp.git
    GIT_TAG 20230802.0 # Matches the version in co's WORKSPACE
    # Pass architecture settings to Abseil's CMake build
    CMAKE_ARGS
        CMAKE_OSX_ARCHITECTURES="${CMAKE_OSX_ARCHITECTURES}"
)
FetchContent_MakeAvailable(abseil)
# Abseil provides targets like absl::base, absl::strings, etc.

# --- External Dependency: Googletest ---
# Fetches Googletest and integrates its CMake build
if (CO_ENABLE_TESTING)
    FetchContent_Declare(
        googletest
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG v1.17.0 # Matches the version in your WORKSPACE
        # Pass architecture settings to Googletest's CMake build
        CMAKE_ARGS
            CMAKE_OSX_ARCHITECTURES="${CMAKE_OSX_ARCHITECTURES}"
    )
    FetchContent_MakeAvailable(googletest)
endif()
# Googletest provides targets like gtest_main, gtest, gmock.

# --- co Library Target ---
# Corresponds to //:co in Bazel
set(TGT_CO co)
add_library(${TGT_CO} STATIC
    co/coroutine.cc
    co/coroutine.h
    co/bitset.h
    co/context.S
    co/context.h
    co/detect_sanitizers.h
)

target_compile_features(${TGT_CO} PUBLIC c_std_17)
set_target_properties(${TGT_CO} PROPERTIES
        C_STANDARD_REQUIRED ON
        C_EXTENSIONS OFF
)
target_compile_features(${TGT_CO} PRIVATE cxx_std_17)
set_target_properties(${TGT_CO} PROPERTIES
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS OFF
)

# Add common compile options (from co/BUILD)
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    target_compile_options(${TGT_CO} PRIVATE -Wall)
    target_compile_options(${TGT_CO} PRIVATE -Wextra)
    target_compile_options(${TGT_CO} PRIVATE -Wpedantic)
endif()

# Headers use "co/coroutine.h" style includes, so we add the project root.
target_include_directories(${TGT_CO} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# Link against Abseil dependencies
target_link_libraries(${TGT_CO} PUBLIC
    absl::base
    absl::synchronization # Used by mutex, condition, etc.
    absl::time # Used by channel, future (for timeouts)
)

# --- co_cpp20 Library Target ---
# Standalone C++20 coroutine library (no dependency on 'co' or Abseil)
set(TGT_CO_CPP20 co_cpp20)
add_library(${TGT_CO_CPP20} STATIC
    co/coroutine_cpp20.cc
    co/coroutine_cpp20.h
)

target_compile_features(${TGT_CO_CPP20} PUBLIC c_std_17)
set_target_properties(${TGT_CO_CPP20} PROPERTIES
        C_STANDARD_REQUIRED ON
        C_EXTENSIONS OFF
)
target_compile_features(${TGT_CO_CPP20} PRIVATE cxx_std_20)
set_target_properties(${TGT_CO_CPP20} PROPERTIES
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS OFF
)

# Add common compile options (from co/BUILD)
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    target_compile_options(${TGT_CO_CPP20} PRIVATE -Wall)
    target_compile_options(${TGT_CO_CPP20} PRIVATE -Wextra)
    target_compile_options(${TGT_CO_CPP20} PRIVATE -Wpedantic)
endif()

target_include_directories(${TGT_CO_CPP20} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# --- Tests ---
if (CO_ENABLE_TESTING)
    enable_testing()

    # C++17 coroutine tests
    set(TGT_COROUTINES_TEST coroutines_test)
    add_executable(${TGT_COROUTINES_TEST}
        co/coroutines_test.cc
    )

    target_compile_features(${TGT_COROUTINES_TEST} PUBLIC c_std_17)
    set_target_properties(${TGT_COROUTINES_TEST} PROPERTIES
            C_STANDARD_REQUIRED ON
            C_EXTENSIONS OFF
    )
    target_compile_features(${TGT_COROUTINES_TEST} PRIVATE cxx_std_20)
    set_target_properties(${TGT_COROUTINES_TEST} PROPERTIES
            CXX_STANDARD_REQUIRED ON
            CXX_EXTENSIONS OFF
    )

    # Add common compile options (from co/BUILD)
    if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
        target_compile_options(${TGT_COROUTINES_TEST} PRIVATE -Wall)
        target_compile_options(${TGT_COROUTINES_TEST} PRIVATE -Wextra)
        target_compile_options(${TGT_COROUTINES_TEST} PRIVATE -Wpedantic)
    endif()

    target_link_libraries(${TGT_COROUTINES_TEST} PRIVATE
        ${TGT_CO}
        absl::status
        gtest_main
    )
    add_test(NAME ${TGT_COROUTINES_TEST} COMMAND ${TGT_COROUTINES_TEST})

    # C++20 coroutine tests
    set(TGT_TEST_CPP20 test_cpp20)
    add_executable(${TGT_TEST_CPP20}
        co/test_cpp20.cc
    )
    target_link_libraries(${TGT_TEST_CPP20} PRIVATE
        ${TGT_CO_CPP20}
        gtest_main
    )
    set_target_properties(${TGT_TEST_CPP20} PROPERTIES
        CXX_STANDARD 20
        CXX_STANDARD_REQUIRED ON
    )
    add_test(NAME ${TGT_TEST_CPP20} COMMAND ${TGT_TEST_CPP20})
endif()