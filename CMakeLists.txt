cmake_minimum_required(VERSION 3.15)
project(sparsehash VERSION 2.0.4 LANGUAGES C CXX)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(CheckIncludeFileCXX)
include(CheckCXXSymbolExists)
include(CheckTypeSize)
include(CheckCXXSourceCompiles)
include(CTest)

# ---------------------------------------------------------------------------
# Options
# ---------------------------------------------------------------------------
option(SPARSEHASH_BUILD_TESTS "Build sparsehash unit tests" ${BUILD_TESTING})
option(SPARSEHASH_BUILD_BENCHMARKS "Build sparsehash benchmarks (time_hash_map)" OFF)
option(SPARSEHASH_INSTALL "Generate install targets for sparsehash" ON)
set(SPARSEHASH_GOOGLE_NAMESPACE "google" CACHE STRING "Namespace for Google classes")

# ---------------------------------------------------------------------------
# Configuration Checks
# ---------------------------------------------------------------------------
# Standard C / C++ headers
check_include_file_cxx("stdint.h" HAVE_STDINT_H)
check_include_file_cxx("sys/types.h" HAVE_SYS_TYPES_H)
check_include_file_cxx("inttypes.h" HAVE_INTTYPES_H)
check_include_file_cxx("stdio.h" HAVE_STDIO_H)
check_include_file_cxx("stdlib.h" HAVE_STDLIB_H)
check_include_file_cxx("string.h" HAVE_STRING_H)
check_include_file_cxx("strings.h" HAVE_STRINGS_H)
check_include_file_cxx("sys/stat.h" HAVE_SYS_STAT_H)
check_include_file_cxx("unistd.h" HAVE_UNISTD_H)
check_include_file_cxx("sys/resource.h" HAVE_SYS_RESOURCE_H)
check_include_file_cxx("sys/time.h" HAVE_SYS_TIME_H)
check_include_file_cxx("sys/utsname.h" HAVE_SYS_UTSNAME_H)
check_include_file_cxx("google/malloc_extension.h" HAVE_GOOGLE_MALLOC_EXTENSION_H)

# Type checks
set(CMAKE_EXTRA_INCLUDE_FILES)
if(HAVE_STDINT_H)
    list(APPEND CMAKE_EXTRA_INCLUDE_FILES "stdint.h")
endif()
if(HAVE_SYS_TYPES_H)
    list(APPEND CMAKE_EXTRA_INCLUDE_FILES "sys/types.h")
endif()
if(HAVE_INTTYPES_H)
    list(APPEND CMAKE_EXTRA_INCLUDE_FILES "inttypes.h")
endif()

check_type_size("uint16_t" UINT16_T)
if(HAVE_UINT16_T)
    set(HAVE_UINT16_T 1)
endif()

check_type_size("u_int16_t" U_INT16_T)
if(HAVE_U_INT16_T)
    set(HAVE_U_INT16_T 1)
endif()

check_type_size("__uint16" __UINT16)
if(HAVE___UINT16)
    set(HAVE___UINT16 1)
endif()

check_type_size("long long" LONG_LONG)
if(HAVE_LONG_LONG)
    set(HAVE_LONG_LONG 1)
endif()

# Functions
check_cxx_symbol_exists(memcpy "string.h;cstring" HAVE_MEMCPY)
check_cxx_symbol_exists(memmove "string.h;cstring" HAVE_MEMMOVE)

# Standard C headers
set(STDC_HEADERS 1)
set(HAVE_NAMESPACES 1)

# Hash function and map headers detection
check_cxx_source_compiles("
    #include <functional>
    int main() {
        int x = std::hash<int>()(5);
        (void)x;
        return 0;
    }
" SPARSEHASH_HAVE_STD_HASH_FUNCTIONAL)

check_cxx_source_compiles("
    #include <unordered_map>
    int main() {
        const std::unordered_map<int, int> t;
        return t.find(5) == t.end();
    }
" SPARSEHASH_HAVE_STD_UNORDERED_MAP)

if(SPARSEHASH_HAVE_STD_HASH_FUNCTIONAL)
    set(HASH_FUN_H "<functional>")
    set(HASH_NAMESPACE "std")
    set(SPARSEHASH_HASH "HASH_NAMESPACE::hash")
    set(SPARSEHASH_HASH_NO_NAMESPACE "hash")
elseif(MSVC)
    set(HASH_FUN_H "<hash_map>")
    set(HASH_NAMESPACE "stdext")
    set(SPARSEHASH_HASH "HASH_NAMESPACE::hash_compare")
    set(SPARSEHASH_HASH_NO_NAMESPACE "hash_compare")
else()
    # Check tr1 / __gnu_cxx fallbacks
    check_cxx_source_compiles("
        #include <tr1/functional>
        int main() {
            int x = std::tr1::hash<int>()(5);
            (void)x;
            return 0;
        }
    " SPARSEHASH_HAVE_TR1_HASH)
    if(SPARSEHASH_HAVE_TR1_HASH)
        set(HASH_FUN_H "<tr1/functional>")
        set(HASH_NAMESPACE "std::tr1")
        set(SPARSEHASH_HASH "HASH_NAMESPACE::hash")
        set(SPARSEHASH_HASH_NO_NAMESPACE "hash")
    else()
        set(HASH_FUN_H "<ext/hash_fun.h>")
        set(HASH_NAMESPACE "__gnu_cxx")
        set(SPARSEHASH_HASH "HASH_NAMESPACE::hash")
        set(SPARSEHASH_HASH_NO_NAMESPACE "hash")
    endif()
endif()

if(SPARSEHASH_HAVE_STD_UNORDERED_MAP)
    set(HASH_MAP_H "<unordered_map>")
    set(HASH_SET_H "<unordered_set>")
    set(HAVE_UNORDERED_MAP 1)
    set(HAVE_HASH_MAP 1)
    set(HAVE_HASH_SET 1)
elseif(MSVC)
    set(HASH_MAP_H "<hash_map>")
    set(HASH_SET_H "<hash_set>")
    set(HAVE_HASH_MAP 1)
    set(HAVE_HASH_SET 1)
endif()

# Namespace configuration
if(SPARSEHASH_GOOGLE_NAMESPACE)
    set(GOOGLE_NAMESPACE "::${SPARSEHASH_GOOGLE_NAMESPACE}")
    set(_START_GOOGLE_NAMESPACE_ "namespace ${SPARSEHASH_GOOGLE_NAMESPACE} {")
    set(_END_GOOGLE_NAMESPACE_ "}")
else()
    set(GOOGLE_NAMESPACE "")
    set(_START_GOOGLE_NAMESPACE_ "")
    set(_END_GOOGLE_NAMESPACE_ "")
endif()

# Package metadata
set(PACKAGE "${PROJECT_NAME}")
set(PACKAGE_NAME "${PROJECT_NAME}")
set(PACKAGE_VERSION "${PROJECT_VERSION}")
set(PACKAGE_STRING "${PROJECT_NAME} ${PROJECT_VERSION}")
set(PACKAGE_TARNAME "${PROJECT_NAME}")
set(PACKAGE_BUGREPORT "google-sparsehash@googlegroups.com")
set(PACKAGE_URL "https://github.com/sparsehash/sparsehash")
set(VERSION "${PROJECT_VERSION}")

# ---------------------------------------------------------------------------
# Generated Headers
# ---------------------------------------------------------------------------
set(SPARSEHASH_GEN_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/include")
set(SPARSEHASH_INTERNAL_CONFIG_DIR "${CMAKE_CURRENT_BINARY_DIR}/internal_config")

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/sparseconfig.h.cmake.in"
    "${SPARSEHASH_GEN_INCLUDE_DIR}/sparsehash/internal/sparseconfig.h"
    @ONLY
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.cmake.in"
    "${SPARSEHASH_INTERNAL_CONFIG_DIR}/config.h"
    @ONLY
)

# ---------------------------------------------------------------------------
# Target Definition
# ---------------------------------------------------------------------------
add_library(sparsehash INTERFACE)
add_library(sparsehash::sparsehash ALIAS sparsehash)

target_compile_features(sparsehash INTERFACE cxx_std_11)

target_include_directories(sparsehash
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${SPARSEHASH_GEN_INCLUDE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# ---------------------------------------------------------------------------
# Tests
# ---------------------------------------------------------------------------
if(SPARSEHASH_BUILD_TESTS)
    enable_testing()

    set(SPARSEHASH_TEST_TARGETS
        template_util_unittest
        type_traits_unittest
        libc_allocator_with_realloc_test
        sparsetable_unittest
        hashtable_test
        simple_test
        simple_compat_test
    )

    set(TEST_PLATFORM_SOURCES "")
    if(WIN32)
        list(APPEND TEST_PLATFORM_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/windows/port.cc")
    endif()

    foreach(test_name IN LISTS SPARSEHASH_TEST_TARGETS)
        add_executable(${test_name}
            "src/${test_name}.cc"
            ${TEST_PLATFORM_SOURCES}
        )
        target_link_libraries(${test_name} PRIVATE sparsehash)
        target_include_directories(${test_name} PRIVATE
            "${SPARSEHASH_INTERNAL_CONFIG_DIR}"
            $<$<BOOL:${WIN32}>:${CMAKE_CURRENT_SOURCE_DIR}/src/windows>
        )
        if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
            target_compile_options(${test_name} PRIVATE
                -Wall -Wextra -Wwrite-strings -Woverloaded-virtual
            )
        endif()
        add_test(NAME ${test_name} COMMAND ${test_name})
    endforeach()
endif()

# ---------------------------------------------------------------------------
# Benchmarks / Programs
# ---------------------------------------------------------------------------
if(SPARSEHASH_BUILD_BENCHMARKS)
    set(TIME_HASH_MAP_SOURCES
        "src/time_hash_map.cc"
    )
    if(WIN32)
        list(APPEND TIME_HASH_MAP_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/windows/port.cc")
    endif()

    add_executable(time_hash_map ${TIME_HASH_MAP_SOURCES})
    target_link_libraries(time_hash_map PRIVATE sparsehash)
    target_include_directories(time_hash_map PRIVATE
        "${SPARSEHASH_INTERNAL_CONFIG_DIR}"
        $<$<BOOL:${WIN32}>:${CMAKE_CURRENT_SOURCE_DIR}/src/windows>
    )

    if(HAVE_GOOGLE_MALLOC_EXTENSION_H)
        find_library(TCMALLOC_LIB NAMES tcmalloc)
        if(TCMALLOC_LIB)
            target_link_libraries(time_hash_map PRIVATE ${TCMALLOC_LIB})
        endif()
    endif()

    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        target_compile_options(time_hash_map PRIVATE
            -Wall -Wextra -Wwrite-strings -Woverloaded-virtual
        )
    endif()
endif()

# ---------------------------------------------------------------------------
# Installation and Export
# ---------------------------------------------------------------------------
if(SPARSEHASH_INSTALL)
    install(
        TARGETS sparsehash
        EXPORT sparsehashTargets
        INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    )

    install(
        DIRECTORY src/sparsehash
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    )

    install(
        DIRECTORY src/google
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    )

    install(
        FILES "${SPARSEHASH_GEN_INCLUDE_DIR}/sparsehash/internal/sparseconfig.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/sparsehash/internal"
    )

    # CMake package files
    set(SPARSEHASH_CMAKECONFIG_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/sparsehash")

    install(
        EXPORT sparsehashTargets
        FILE sparsehashTargets.cmake
        NAMESPACE sparsehash::
        DESTINATION "${SPARSEHASH_CMAKECONFIG_DIR}"
    )

    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/sparsehashConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    configure_package_config_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/sparsehashConfig.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/sparsehashConfig.cmake"
        INSTALL_DESTINATION "${SPARSEHASH_CMAKECONFIG_DIR}"
    )

    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/sparsehashConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/sparsehashConfigVersion.cmake"
        DESTINATION "${SPARSEHASH_CMAKECONFIG_DIR}"
    )

    # For case-insensitive or SparseHash find_package compatibility
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/SparseHashConfig.cmake"
        "include(\"\${CMAKE_CURRENT_LIST_DIR}/../sparsehash/sparsehashConfig.cmake\")\n"
    )
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/SparseHashConfigVersion.cmake"
        "include(\"\${CMAKE_CURRENT_LIST_DIR}/../sparsehash/sparsehashConfigVersion.cmake\")\n"
    )

    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/SparseHashConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/SparseHashConfigVersion.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/SparseHash"
    )

    # pkg-config files
    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/libsparsehash.pc.in"
        "${CMAKE_CURRENT_BINARY_DIR}/libsparsehash.pc"
        @ONLY
    )
    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/libsparsehash.pc.in"
        "${CMAKE_CURRENT_BINARY_DIR}/sparsehash.pc"
        @ONLY
    )

    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/libsparsehash.pc"
            "${CMAKE_CURRENT_BINARY_DIR}/sparsehash.pc"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
    )

    # Documentation
    install(
        FILES
            AUTHORS
            COPYING
            ChangeLog
            NEWS
            README
            TODO
        DESTINATION "${CMAKE_INSTALL_DOCDIR}"
        OPTIONAL
    )

    install(
        DIRECTORY doc/
        DESTINATION "${CMAKE_INSTALL_DOCDIR}/html"
        OPTIONAL
    )

    # Export for build tree usage
    export(
        EXPORT sparsehashTargets
        FILE "${CMAKE_CURRENT_BINARY_DIR}/sparsehashTargets.cmake"
        NAMESPACE sparsehash::
    )
endif()
