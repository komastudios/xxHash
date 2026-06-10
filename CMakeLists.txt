cmake_minimum_required(VERSION 3.21)

file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/xxhash.h" XXHASH_VERSION_MAJOR REGEX "^#define XXH_VERSION_MAJOR +([0-9]+) *$")
string(REGEX REPLACE "^#define XXH_VERSION_MAJOR +([0-9]+) *$" "\\1" XXHASH_VERSION_MAJOR "${XXHASH_VERSION_MAJOR}")
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/xxhash.h" XXHASH_VERSION_MINOR REGEX "^#define XXH_VERSION_MINOR +([0-9]+) *$")
string(REGEX REPLACE "^#define XXH_VERSION_MINOR +([0-9]+) *$" "\\1" XXHASH_VERSION_MINOR "${XXHASH_VERSION_MINOR}")
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/xxhash.h" XXHASH_VERSION_RELEASE REGEX "^#define XXH_VERSION_RELEASE +([0-9]+) *$")
string(REGEX REPLACE "^#define XXH_VERSION_RELEASE +([0-9]+) *$" "\\1" XXHASH_VERSION_RELEASE "${XXHASH_VERSION_RELEASE}")
set(XXHASH_VERSION_STRING "${XXHASH_VERSION_MAJOR}.${XXHASH_VERSION_MINOR}.${XXHASH_VERSION_RELEASE}")

project(xxhash
    VERSION ${XXHASH_VERSION_STRING}
    LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED True)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

option(BUILD_SHARED "Build shared library" ON)
option(USE_MSVC_STATIC_RUNTIME "Use MSVC static runtime" OFF)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    add_compile_options("-ffile-prefix-map=${CMAKE_CURRENT_SOURCE_DIR}=.")
    add_compile_options("-fdebug-prefix-map=${CMAKE_CURRENT_SOURCE_DIR}=.")
endif()

if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
    add_compile_options("/Brepro")
    add_link_options("/Brepro")
    add_compile_options("/pathmap:${CMAKE_CURRENT_SOURCE_DIR}=.")
    # add_link_options only reaches link.exe (executables + shared libs).
    # Static archives go through lib.exe, which needs /Brepro via the
    # separate CMAKE_STATIC_LINKER_FLAGS variable — otherwise the .lib
    # picks up a per-run timestamp in its archive header.
    string(APPEND CMAKE_STATIC_LINKER_FLAGS " /Brepro")
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Android")
    set(BUILD_SHARED OFF)

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -g1 -DNDEBUG")

    add_compile_options(-fsigned-char)
    add_link_options("-Wl,-z,max-page-size=16384")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "^(iOS|tvOS)$")
    set(BUILD_SHARED OFF)

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -g1 -DNDEBUG")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
    set(BUILD_SHARED OFF)

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -g1 -DNDEBUG")
endif()

if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
    if(USE_MSVC_STATIC_RUNTIME)
        message(STATUS "Using MSVC static runtime")
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    else()
        message(STATUS "Using MSVC dynamic runtime")
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
    endif()
endif()

set(LIB_SOURCE_FILES xxhash.c)

function(add_xxhash_library TARGET_NAME LIB_TYPE)
    add_library(${TARGET_NAME} ${LIB_TYPE} ${LIB_SOURCE_FILES})
    target_include_directories(${TARGET_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
    set_property(TARGET ${TARGET_NAME} PROPERTY C_VISIBILITY_PRESET hidden)

    if(LIB_TYPE STREQUAL "STATIC")
        set_target_properties(${TARGET_NAME} PROPERTIES OUTPUT_NAME "xxhash-static")
    else()
        set_target_properties(${TARGET_NAME} PROPERTIES
            OUTPUT_NAME "xxhash")
        target_compile_definitions(${TARGET_NAME} PRIVATE XXH_EXPORT)
        if(WIN32)
            target_compile_definitions(${TARGET_NAME} PUBLIC XXH_IMPORT)
        endif()
    endif()

    install(TARGETS ${TARGET_NAME}
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin)
endfunction()

add_xxhash_library(xxhash-static STATIC)

install(FILES xxhash.h xxh3.h
    DESTINATION include)

if(BUILD_SHARED)
    add_xxhash_library(xxhash SHARED)
endif()

include(CTest)
if(BUILD_TESTING AND NOT CMAKE_CROSSCOMPILING)
    add_executable(test_xxhash tests/test_xxhash.c)
    target_include_directories(test_xxhash PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
    target_link_libraries(test_xxhash PRIVATE xxhash-static)
    add_test(NAME test_xxhash COMMAND test_xxhash)
endif()
